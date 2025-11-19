# dex
1
"""
WSOL-USDC-WSOL 简化生产循环脚本（Step 1）- 优化版
===============================================

功能：
- 单一路径循环套利：WSOL → USDC → WSOL
- 固定仓位（默认 0.2 WSOL），无限循环
- 🔥 智能限流优化：Quote 立即执行捕捉行情，Swap 前等1秒，下轮等2秒
- 精准优化：跳过 DFlow 的 wrap_sol setup 指令（用户已持有 WSOL）
- 实时统计：累计盈亏（WSOL 计价）、胜率、平均盈利
- 自动暂停：22 轮后暂停 2 分钟，避免 429 封禁

准备步骤：
1) 配置环境变量（在项目根目录创建 .env 文件）：
   SOLANA_RPC_URL=https://mainnet.helius-rpc.com/?api-key=YOUR_KEY
   SOLANA_PRIVATE_KEY=[...] 或 base58 字符串

2) 确保钱包有足够余额：
   - 至少 0.5 WSOL（建议 2 WSOL 用于生产）
   - 少量 SOL 用于交易费用（~0.01 SOL）

3) 安装依赖（如未安装）：
   pip install solders solana aiohttp python-dotenv

运行：
   python wsol_usdc_production_simple.py

后台运行（Linux）：
   nohup python3 wsol_usdc_production_simple.py > output.log 2>&1 &
   tail -f output.log

停止：
   按 Ctrl+C 安全退出并查看完整统计
   或 pkill -f wsol_usdc_production_simple.py

配置参数（可在下方代码中调整）：
   AMOUNT_LAMPORTS = 200_000_000     # 固定仓位：0.2 WSOL
   SLIPPAGE_BPS = 0                  # 滑点容忍：0 = 无限制
   PRIORITY_FEE_LAMPORTS = 0         # 优先费用：0 = 仅签名费
   MIN_PROFIT_BPS = -50              # 最小净利润（BPS），低于此值跳过交易
   QUOTE_GAP_SEC = 0                 # Quote1 和 Quote2 之间（0 = 立即执行）
   SWAP_WAIT_SEC = 1                 # Swap API 前等待（避免限流）
   NEXT_ROUND_WAIT_SEC = 2           # 下一轮前等待
   MAX_ROUNDS_BEFORE_PAUSE = 22      # 执行 22 轮后自动暂停
   PAUSE_DURATION_SEC = 120          # 暂停时长（秒）
"""
import asyncio
import json
import os
import base58
import base64
from typing import List

from dotenv import load_dotenv

from solders.keypair import Keypair
from solders.pubkey import Pubkey
from solders.instruction import Instruction, AccountMeta
from solders.transaction import VersionedTransaction
from solders.message import MessageV0
from solders.address_lookup_table_account import AddressLookupTableAccount
from solders.compute_budget import set_compute_unit_limit
from solana.rpc.async_api import AsyncClient
from solana.rpc.commitment import Confirmed, Finalized
from solana.rpc.types import TxOpts

import aiohttp

import logging

logger = logging.getLogger("wsol_usdc_prod")
logger.setLevel(logging.INFO)
_console = logging.StreamHandler()
_console.setFormatter(logging.Formatter('%(message)s'))
logger.addHandler(_console)

# 加载 .env
load_dotenv()
RPC_URL = os.getenv("SOLANA_RPC_URL", "https://api.mainnet-beta.solana.com")
WALLET_PRIVATE_KEY = os.getenv("SOLANA_PRIVATE_KEY")

# Token & Program IDs
WSOL = "So11111111111111111111111111111111111111112"
USDC = "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"

DFLOW_PROGRAM_ID = "DF1ow4tspfHX9JwWJsAb9epbkA8hmpSEAtxXy1V27QBH"
WSOL_MINT = WSOL
TOKEN_PROGRAM_ID = "TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA"
ASSOCIATED_TOKEN_PROGRAM_ID = "ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL"
SYSTEM_PROGRAM_ID = "11111111111111111111111111111111"

# ========================================
# 🔥 自定义交易对配置（循环套利）
# ========================================
# 格式: TOKEN_A → TOKEN_B → TOKEN_A
# 修改下面三个参数即可切换交易对

INPUT_TOKEN = WSOL              # 起始 Token（输入）
MIDDLE_TOKEN = USDC             # 中间 Token
OUTPUT_TOKEN = WSOL             # 结束 Token（应该与 INPUT_TOKEN 相同）

# Token 信息（用于显示，可选）
INPUT_TOKEN_NAME = "WSOL"       # 起始 Token 名称
MIDDLE_TOKEN_NAME = "USDC"      # 中间 Token 名称
INPUT_TOKEN_DECIMALS = 9        # 起始 Token 精度（SOL/WSOL = 9）
MIDDLE_TOKEN_DECIMALS = 6       # 中间 Token 精度（USDC = 6）

# 示例：测试其他交易对
# ------------------------------
# WSOL → BONK → WSOL:
#   INPUT_TOKEN = WSOL
#   MIDDLE_TOKEN = "DezXAZ8z7PnrnRJjz3wXBoRgixCa6xjnB7YaB1pPB263"  # BONK
#   INPUT_TOKEN_NAME = "WSOL"
#   MIDDLE_TOKEN_NAME = "BONK"
#   INPUT_TOKEN_DECIMALS = 9
#   MIDDLE_TOKEN_DECIMALS = 5
#
# WSOL → RAY → WSOL:
#   INPUT_TOKEN = WSOL
#   MIDDLE_TOKEN = "4k3Dyjzvzp8eMZWUXbBCjEvwSkkk59S5iCNLY3QrkX6R"  # RAY
#   MIDDLE_TOKEN_NAME = "RAY"
#   MIDDLE_TOKEN_DECIMALS = 6
#
# WSOL → JUP → WSOL:
#   INPUT_TOKEN = WSOL
#   MIDDLE_TOKEN = "JUPyiwrYJFskUPiHa7hkeR8VUtAeFoSYbKedZNsDvCN"  # JUP
#   MIDDLE_TOKEN_NAME = "JUP"
#   MIDDLE_TOKEN_DECIMALS = 6
# ========================================

# 交易参数
AMOUNT_LAMPORTS = 200_000_000  # 0.1 WSOL 固定仓位（Step1）
SLIPPAGE_BPS = 0               # 0 = 允许任意滑点（参考成功交易设置）
PRIORITY_FEE_LAMPORTS = 0      # 不设置优先费（与成功交易保持一致）

# 利润阈值（重要！）
MIN_PROFIT_BPS = -50           # 最小净利润（BPS），低于此值跳过交易
                               # -50 BPS = -0.5% (允许小亏损，因为可能有统计误差)
                               # 0 BPS = 保本（推荐）
                               # 5 BPS = 0.05% 最小盈利（保守）
                               # 10 BPS = 0.1% 最小盈利（激进）

# 路径限制
MAX_ROUTE_HOPS = 3
PREFERRED_ROUTE_HOPS = 2

# 节流优化（精细控制）
QUOTE_GAP_SEC = 0           # Quote1 和 Quote2 之间（不限速，立即执行）
SWAP_WAIT_SEC = 1           # Swap API 调用前等待（避免限流）
NEXT_ROUND_WAIT_SEC = 2     # 下一轮开始前等待
LOOP_SLEEP_SEC = 10         # 每轮之间（跳过或失败时）

# 🔥 自动暂停机制（避免 429 频繁限流）
MAX_ROUNDS_BEFORE_PAUSE = 22    # 执行 22 轮后自动暂停
PAUSE_DURATION_SEC = 120        # 暂停 2 分钟（避免 429 10分钟封禁）

# 估算签名费（保守值，主网 ~5000 lamports 左右）
SIG_FEE_EST = 5_000

# 日志控制
SHOW_ALT_DETAILS = False  # 是否显示 ALT 详细加载信息（生产环境建议 False）
SHOW_ERROR_TRACE = False  # 是否显示完整错误堆栈（调试时设为 True）


def is_dflow_wrap_sol_instruction(inst_data: dict) -> bool:
    """
    精确识别 DFlow 的 wrap_sol 指令（恰好 6 个账户 + 账户/程序严格匹配）。
    仅跳过该指令，不影响其它 setup/cleanup。
    """
    program_id = inst_data.get("programId", "")
    accounts = inst_data.get("accounts", [])

    if program_id != DFLOW_PROGRAM_ID:
        return False
    if len(accounts) != 6:
        return False

    try:
        if accounts[2]["pubkey"] != WSOL_MINT:
            return False
        if accounts[3]["pubkey"] != TOKEN_PROGRAM_ID:
            return False
        if accounts[4]["pubkey"] != ASSOCIATED_TOKEN_PROGRAM_ID:
            return False
        if accounts[5]["pubkey"] != SYSTEM_PROGRAM_ID:
            return False
        if not accounts[0]["isSigner"] or not accounts[0]["isWritable"]:
            return False
        if not accounts[1]["isWritable"]:
            return False
        logger.info("   ⏭️  检测到 DFlow wrap_sol 指令，已跳过（用户已有 WSOL）")
        return True
    except (KeyError, IndexError):
        return False


def parse_account_meta(account: dict) -> AccountMeta:
    return AccountMeta(
        pubkey=Pubkey.from_string(account["pubkey"]),
        is_signer=account["isSigner"],
        is_writable=account["isWritable"],
    )


def parse_instruction(ix: dict) -> Instruction:
    return Instruction(
        program_id=Pubkey.from_string(ix["programId"]),
        accounts=[parse_account_meta(acc) for acc in ix["accounts"]],
        data=base64.b64decode(ix["data"]),
    )


async def try_fetch_alts(rpc_client: AsyncClient, alt_addresses: List[str]) -> List[AddressLookupTableAccount]:
    """尝试加载 Address Lookup Tables，失败不影响交易（交易会变大）"""
    if not alt_addresses:
        return []

    if SHOW_ALT_DETAILS:
        logger.info("🔍 检测 ALT 可用性 ...")
    
    alt_accounts: List[AddressLookupTableAccount] = []

    for i, alt_address in enumerate(alt_addresses, 1):
        try:
            pubkey = Pubkey.from_string(alt_address)
            response = await rpc_client.get_account_info(pubkey, commitment=Confirmed)
            if response.value is None:
                if SHOW_ALT_DETAILS:
                    logger.warning(f"   [{i}/{len(alt_addresses)}] ❌ ALT 不存在或已过期: {alt_address[:8]}...")
                continue

            account_data = bytes(response.value.data)
            if len(account_data) < 56:
                if SHOW_ALT_DETAILS:
                    logger.warning(f"   [{i}/{len(alt_addresses)}] ❌ ALT 数据太短: {alt_address[:8]}...")
                continue

            num_addresses = (len(account_data) - 56) // 32
            addresses = []
            for j in range(num_addresses):
                offset = 56 + j * 32
                addr_bytes = account_data[offset:offset+32]
                addresses.append(Pubkey(addr_bytes))

            alt_accounts.append(AddressLookupTableAccount(key=pubkey, addresses=addresses))
            if SHOW_ALT_DETAILS:
                logger.info(f"   [{i}/{len(alt_addresses)}] ✅ {alt_address[:8]}... ({len(addresses)} addresses)")
        except Exception as e:
            if SHOW_ALT_DETAILS:
                logger.warning(f"   [{i}/{len(alt_addresses)}] ❌ 解析失败 {alt_address[:8]}...: {e}")
            continue

    if SHOW_ALT_DETAILS:
        logger.info(f"📊 ALT 可用: {len(alt_accounts)}/{len(alt_addresses)}\n")
    
    return alt_accounts


async def get_quote(input_mint: str, output_mint: str, amount: int) -> dict:
    """调用 DFlow Quote API 获取报价"""
    url = "https://quote-api.dflow.net/quote"
    params = {
        "inputMint": input_mint,
        "outputMint": output_mint,
        "amount": str(amount),
        "slippageBps": str(SLIPPAGE_BPS),
    }
    try:
        async with aiohttp.ClientSession() as session:
            async with session.get(url, params=params, timeout=aiohttp.ClientTimeout(total=10)) as resp:
                if resp.status != 200:
                    text = await resp.text()
                    raise Exception(f"Quote API 返回错误 {resp.status}: {text[:200]}")
                return await resp.json()
    except asyncio.TimeoutError:
        raise Exception("Quote API 请求超时（10秒）")
    except aiohttp.ClientError as e:
        raise Exception(f"Quote API 网络错误: {e}")


async def get_swap_instructions(quote: dict, user_pubkey: str, priority_fee: int = None) -> dict:
    """调用 DFlow Swap API 获取交易指令"""
    url = "https://quote-api.dflow.net/swap-instructions"
    payload = {"quoteResponse": quote, "userPublicKey": user_pubkey}
    if priority_fee is not None:
        payload["prioritizationFeeLamports"] = priority_fee
    
    try:
        async with aiohttp.ClientSession() as session:
            async with session.post(url, json=payload, timeout=aiohttp.ClientTimeout(total=10)) as resp:
                if resp.status != 200:
                    text = await resp.text()
                    raise Exception(f"Swap API 返回错误 {resp.status}: {text[:200]}")
                return await resp.json()
    except asyncio.TimeoutError:
        raise Exception("Swap API 请求超时（10秒）")
    except aiohttp.ClientError as e:
        raise Exception(f"Swap API 网络错误: {e}")


def merge_quotes_wsol_cycle(q1: dict, q2: dict) -> dict:
    """将 Quote1(WSOL→USDC) 与 Quote2(USDC→WSOL) 合并为单个循环 Quote"""
    return {
        "inputMint": q1["inputMint"],
        "inAmount": q1["inAmount"],
        "outputMint": q2["outputMint"],
        # 🔥 关键：将 outAmount 也设置为 "0" 以禁用 DFlow 的自动 quoted_out_amount 生成
        "outAmount": "0",
        "routePlan": q1["routePlan"] + q2["routePlan"],
        # 🔥 关键修改：设置为 "0" 以完全禁用滑点检查（与成功交易保持一致）
        "otherAmountThreshold": "0",
        "minOutAmount": "0",
        "slippageBps": SLIPPAGE_BPS,
        "platformFee": q1.get("platformFee") or q2.get("platformFee"),
        "outTransferFee": q2.get("outTransferFee"),
        "priceImpactPct": str(float(q1.get("priceImpactPct", 0)) + float(q2.get("priceImpactPct", 0))),
        "simulatedComputeUnits": q1.get("simulatedComputeUnits", 0) + q2.get("simulatedComputeUnits", 0),
        "contextSlot": max(q1.get("contextSlot", 0), q2.get("contextSlot", 0)),
        "requestId": f"merged-{q1.get('requestId','unknown')}-{q2.get('requestId','unknown')}"
    }


def calc_profit_ws(in_amount: int, out_amount: int, priority_fee: int = 0, sig_fee: int = SIG_FEE_EST) -> dict:
    """计算 WSOL 计价的盈亏（不含优先费，只算签名费）"""
    gross = out_amount - in_amount
    net = gross - sig_fee  # 只扣除签名费，不扣除优先费
    bps = (net / in_amount) * 10_000
    return {"gross": gross, "net": net, "bps": bps}


async def build_tx(rpc_client: AsyncClient, keypair: Keypair, swap_result: dict) -> VersionedTransaction:
    instructions: List[Instruction] = []

    # 🔥 手动设置 Compute Unit Limit（无优先费，参考成功案例 ~276K-294K）
    compute_unit_limit = 400_000  # 设置 400K 上限（高于成功案例的最高值）
    instructions.append(set_compute_unit_limit(compute_unit_limit))

    # Setup（跳过 wrap_sol）
    for ix in swap_result.get("setupInstructions", []):
        if not is_dflow_wrap_sol_instruction(ix):
            instructions.append(parse_instruction(ix))

    # Swap
    instructions.append(parse_instruction(swap_result["swapInstruction"]))

    # Cleanup（全部保留）
    for ix in swap_result.get("cleanupInstructions", []):
        instructions.append(parse_instruction(ix))

    # ALT（如有）
    alt_addresses = swap_result.get("addressLookupTableAddresses", [])
    alt_accounts = await try_fetch_alts(rpc_client, alt_addresses) if alt_addresses else []

    # Blockhash & Message
    bh_resp = await rpc_client.get_latest_blockhash(commitment=Finalized)
    recent_blockhash = bh_resp.value.blockhash

    message = MessageV0.try_compile(
        payer=keypair.pubkey(),
        instructions=instructions,
        address_lookup_table_accounts=alt_accounts,
        recent_blockhash=recent_blockhash,
    )
    tx = VersionedTransaction(message, [keypair])
    return tx


async def send_and_confirm(rpc_client: AsyncClient, tx: VersionedTransaction) -> str:
    opts = TxOpts(skip_preflight=True, preflight_commitment=Confirmed, max_retries=3)
    sig_resp = await rpc_client.send_transaction(tx, opts=opts)
    signature = sig_resp.value

    await rpc_client.confirm_transaction(signature, commitment=Confirmed)
    try:
        await asyncio.wait_for(rpc_client.confirm_transaction(signature, commitment=Finalized), timeout=60)
    except asyncio.TimeoutError:
        logger.warning("⚠️  Finalized 超时，可能已成功")
    return signature


async def main_loop(stats: dict):
    """主循环：持续执行 WSOL-USDC-WSOL 套利"""
    # 加载钱包
    if not WALLET_PRIVATE_KEY:
        raise RuntimeError("未设置 SOLANA_PRIVATE_KEY 环境变量")

    private_key_str = WALLET_PRIVATE_KEY.strip().strip('"').strip("'")
    if private_key_str.startswith('['):
        private_key_data = json.loads(private_key_str)
        keypair = Keypair.from_bytes(bytes(private_key_data))
    else:
        private_key_bytes = base58.b58decode(private_key_str)
        keypair = Keypair.from_bytes(private_key_bytes)

    user_pubkey = str(keypair.pubkey())

    logger.info("=" * 80)
    logger.info(f"🚀 {INPUT_TOKEN_NAME}-{MIDDLE_TOKEN_NAME}-{INPUT_TOKEN_NAME} 循环套利")
    logger.info("=" * 80)
    logger.info(f"RPC: {RPC_URL}")
    logger.info(f"钱包: {user_pubkey}")
    logger.info(f"交易路径: {INPUT_TOKEN_NAME} → {MIDDLE_TOKEN_NAME} → {INPUT_TOKEN_NAME}")
    logger.info(f"固定仓位: {AMOUNT_LAMPORTS/10**INPUT_TOKEN_DECIMALS:.6f} {INPUT_TOKEN_NAME}")
    logger.info(f"滑点: {SLIPPAGE_BPS} BPS (0 = 无限制)")
    logger.info(f"优先费: 0 lamports（仅签名费）")
    logger.info(f"最小利润阈值: {MIN_PROFIT_BPS} BPS ({MIN_PROFIT_BPS/100}%)")
    logger.info(f"Quote 间隔: {QUOTE_GAP_SEC} 秒, Swap 等待: {SWAP_WAIT_SEC} 秒, 下轮等待: {NEXT_ROUND_WAIT_SEC} 秒")
    logger.info("")

    async with AsyncClient(RPC_URL) as rpc_client:
        # 打印初始余额
        bal_resp = await rpc_client.get_balance(keypair.pubkey())
        initial_balance = bal_resp.value
        logger.info(f"💰 初始余额: {initial_balance/1_000_000_000:.6f} SOL")
        logger.info("")
        logger.info("提示：按 Ctrl+C 安全停止并查看统计")
        logger.info("=" * 80)
        logger.info("")

        while True:
            stats["rounds"] += 1
            round_num = stats["rounds"]
            
            # 🔥 自动暂停机制（避免 429 频繁限流）- 在轮次开始前检查
            if round_num > 1 and (round_num - 1) % MAX_ROUNDS_BEFORE_PAUSE == 0:
                logger.info("=" * 80)
                logger.info(f"⏸️  已完成 {round_num - 1} 轮，自动暂停 {PAUSE_DURATION_SEC} 秒避免 429 限流")
                logger.info(f"   (429 错误会导致 10 分钟封禁，提前暂停 2 分钟可避免)")
                logger.info("=" * 80)
                logger.info("")
                await asyncio.sleep(PAUSE_DURATION_SEC)
            
            logger.info("=" * 80)
            logger.info(f"🔄 第 {round_num} 轮 - {INPUT_TOKEN_NAME} → {MIDDLE_TOKEN_NAME} → {INPUT_TOKEN_NAME}")
            logger.info("=" * 80)

            try:
                # Quote1: INPUT_TOKEN -> MIDDLE_TOKEN（立即执行，不等待）
                logger.info(f"📡 请求 Quote1: {INPUT_TOKEN_NAME} → {MIDDLE_TOKEN_NAME} (金额: {AMOUNT_LAMPORTS/10**INPUT_TOKEN_DECIMALS:.6f} {INPUT_TOKEN_NAME})")
                q1 = await get_quote(INPUT_TOKEN, MIDDLE_TOKEN, AMOUNT_LAMPORTS)
                out1 = int(q1["outAmount"])
                logger.info(f"   ✅ Quote1: 预期得到 {out1/10**MIDDLE_TOKEN_DECIMALS:.6f} {MIDDLE_TOKEN_NAME} (路径: {len(q1['routePlan'])} hops)")

                # Quote2: MIDDLE_TOKEN -> OUTPUT_TOKEN（立即执行，捕捉最新行情）
                if QUOTE_GAP_SEC > 0:
                    await asyncio.sleep(QUOTE_GAP_SEC)
                logger.info(f"📡 请求 Quote2: {MIDDLE_TOKEN_NAME} → {INPUT_TOKEN_NAME} (金额: {out1/10**MIDDLE_TOKEN_DECIMALS:.6f} {MIDDLE_TOKEN_NAME})")
                q2 = await get_quote(MIDDLE_TOKEN, OUTPUT_TOKEN, out1)
                out2 = int(q2["outAmount"])
                logger.info(f"   ✅ Quote2: 预期得到 {out2/10**INPUT_TOKEN_DECIMALS:.6f} {INPUT_TOKEN_NAME} (路径: {len(q2['routePlan'])} hops)")

                # 路径长度检查
                hops_total = len(q1["routePlan"]) + len(q2["routePlan"])
                if hops_total > MAX_ROUTE_HOPS:
                    logger.warning(f"⏭️  路径过长: {hops_total} > {MAX_ROUTE_HOPS}，跳过本轮")
                    stats["skipped_path"] += 1
                    await asyncio.sleep(LOOP_SLEEP_SEC)
                    continue
                elif hops_total <= PREFERRED_ROUTE_HOPS:
                    logger.info(f"✅ 优质路径: {hops_total} hops ≤ {PREFERRED_ROUTE_HOPS}")
                else:
                    logger.info(f"⚠️  复杂路径: {hops_total} hops")

                # 合并 Quote
                logger.info("🔗 合并两个 Quote 为循环路径...")
                merged = merge_quotes_wsol_cycle(q1, q2)

                # 提前计算预期盈亏（用于决策）
                # 注意：我们将 outAmount 设为 "0" 来禁用滑点检查，所以这里使用 q2 的原始值计算
                out_final = int(q2["outAmount"])
                pnl_preview = calc_profit_ws(AMOUNT_LAMPORTS, out_final)  # 不传 priority_fee
                logger.info(f"💡 预期盈亏: gross={pnl_preview['gross']/1_000_000_000:+.6f} WSOL, net={pnl_preview['net']/1_000_000_000:+.6f} WSOL ({pnl_preview['bps']:+.2f} bps)")

                # 🚨 利润判断：低于阈值则跳过交易
                if pnl_preview['bps'] < MIN_PROFIT_BPS:
                    logger.warning(f"⏭️  预期利润过低: {pnl_preview['bps']:.2f} bps < {MIN_PROFIT_BPS} bps，跳过本轮")
                    logger.warning(f"   (净利润: {pnl_preview['net']/1_000_000_000:+.6f} WSOL)")
                    stats["skipped_profit"] += 1
                    await asyncio.sleep(LOOP_SLEEP_SEC)
                    continue
                else:
                    logger.info(f"✅ 利润检查通过: {pnl_preview['bps']:.2f} bps >= {MIN_PROFIT_BPS} bps")

                # Swap API（等待1秒避免限流）
                if SWAP_WAIT_SEC > 0:
                    await asyncio.sleep(SWAP_WAIT_SEC)
                logger.info("📡 请求 Swap Instructions...")
                # 不传递 priority_fee 参数，使用默认值（无优先费）
                swap_result = await get_swap_instructions(merged, user_pubkey, priority_fee=None)
                logger.info(f"   ✅ Swap 指令获取成功")

                # 构建交易
                logger.info("🔨 构建交易...")
                tx = await build_tx(rpc_client, keypair, swap_result)
                tx_size = len(bytes(tx))
                logger.info(f"   📦 交易大小: {tx_size} bytes (限制: 1232)")
                
                if tx_size > 1232:
                    logger.error(f"   ❌ 交易过大: {tx_size} > 1232，跳过本轮")
                    stats["failed"] += 1
                    await asyncio.sleep(LOOP_SLEEP_SEC)
                    continue

                # 发送交易
                logger.info("📤 发送交易到链上...")
                sig = await send_and_confirm(rpc_client, tx)
                logger.info(f"✅ 交易成功确认")
                logger.info(f"   签名: {sig}")
                logger.info(f"   浏览器: https://solscan.io/tx/{sig}")

                # 更新统计（使用预期值，实际可通过解析交易获取准确值）
                stats["success"] += 1
                stats["pnl_net"] += int(pnl_preview["net"])

                # 计算胜率
                win_rate = (stats["success"] / stats["rounds"]) * 100

                logger.info("")
                logger.info(f"💰 本轮盈亏:")
                logger.info(f"   毛利: {pnl_preview['gross']/10**INPUT_TOKEN_DECIMALS:+.6f} {INPUT_TOKEN_NAME}")
                logger.info(f"   净利: {pnl_preview['net']/10**INPUT_TOKEN_DECIMALS:+.6f} {INPUT_TOKEN_NAME} ({pnl_preview['bps']:+.2f} bps)")
                logger.info("")
                logger.info(f"📈 累计统计:")
                logger.info(f"   轮次: {stats['rounds']} (成功: {stats['success']}, 失败: {stats['failed']}, 跳过: {stats['skipped_profit']+stats['skipped_path']})")
                logger.info(f"   胜率: {win_rate:.1f}%")
                logger.info(f"   累计净盈亏: {stats['pnl_net']/10**INPUT_TOKEN_DECIMALS:+.6f} {INPUT_TOKEN_NAME}")
                if stats["success"] > 0:
                    avg = stats["pnl_net"] / stats["success"]
                    logger.info(f"   平均盈利/笔: {avg/10**INPUT_TOKEN_DECIMALS:+.6f} {INPUT_TOKEN_NAME}")
                logger.info("")

            except KeyboardInterrupt:
                raise
            except Exception as e:
                stats["failed"] += 1
                logger.error(f"❌ 本轮失败: {e}")
                
                # 打印错误堆栈（根据配置）
                if SHOW_ERROR_TRACE:
                    import traceback
                    logger.error(traceback.format_exc())
                
                # 限流退避
                msg = str(e)
                if "429" in msg or "rate" in msg.lower() or "too many" in msg.lower():
                    sleep_s = 20  # 遇到限流，休眠更久
                    logger.warning(f"⚠️  检测到 API 限流，休眠 {sleep_s} 秒")
                else:
                    sleep_s = LOOP_SLEEP_SEC
                
                logger.info(f"⏰ {sleep_s} 秒后继续下一轮")
                logger.info("")
                await asyncio.sleep(sleep_s)
                continue

            # 正常轮次间隔（成功执行后等待，准备下一轮）
            if NEXT_ROUND_WAIT_SEC > 0:
                logger.info(f"⏰ 等待 {NEXT_ROUND_WAIT_SEC} 秒后开始下一轮...")
                logger.info("")
                await asyncio.sleep(NEXT_ROUND_WAIT_SEC)


async def main():
    """主入口，处理 KeyboardInterrupt 并打印最终统计"""
    stats = {
        "success": 0, 
        "failed": 0, 
        "skipped_profit": 0,  # 利润过低跳过
        "skipped_path": 0,    # 路径过长跳过
        "pnl_net": 0, 
        "rounds": 0
    }
    try:
        await main_loop(stats)
    except KeyboardInterrupt:
        logger.info("\n" + "=" * 80)
        logger.info("🛑 用户手动停止")
        logger.info("=" * 80)
        logger.info(f"📊 最终统计:")
        logger.info(f"   总轮次: {stats['rounds']}")
        logger.info(f"   ✅ 成功: {stats['success']}")
        logger.info(f"   ❌ 失败: {stats['failed']}")
        logger.info(f"   ⏭️  跳过（利润低）: {stats['skipped_profit']}")
        logger.info(f"   ⏭️  跳过（路径长）: {stats['skipped_path']}")
        total_executed = stats['success'] + stats['failed']
        total_skipped = stats['skipped_profit'] + stats['skipped_path']
        logger.info(f"   执行率: {total_executed}/{stats['rounds']} ({total_executed/stats['rounds']*100 if stats['rounds'] > 0 else 0:.1f}%)")
        win_rate = (stats['success'] / total_executed * 100) if total_executed > 0 else 0
        logger.info(f"   胜率（执行的）: {win_rate:.1f}%")
        logger.info(f"   净盈亏: {stats['pnl_net']/10**INPUT_TOKEN_DECIMALS:+.6f} {INPUT_TOKEN_NAME}")
        if stats['success'] > 0:
            avg_profit = stats['pnl_net'] / stats['success']
            logger.info(f"   平均盈利/笔: {avg_profit/10**INPUT_TOKEN_DECIMALS:+.6f} {INPUT_TOKEN_NAME}")
        logger.info("=" * 80)


if __name__ == "__main__":
    asyncio.run(main())
