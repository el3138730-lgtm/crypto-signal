import requests
import json
import time
import hmac
import base64
from datetime import datetime, timezone
import hashlib

# ========== 从环境变量读取密钥 ==========
import os
API_KEY = os.environ.get('OKX_API_KEY')
SECRET_KEY = os.environ.get('OKX_SECRET_KEY')
PASSPHRASE = os.environ.get('OKX_PASSPHRASE')
SERVERCHAN_KEY = os.environ.get('SERVERCHAN_KEY')

# ========== 风控参数 ==========
TOTAL_CAPITAL = 9.9
MAX_LOSS_RATIO = 0.05
MAX_LOSS_AMOUNT = TOTAL_CAPITAL * MAX_LOSS_RATIO
RSI_OVERSOLD = 30
RSI_PERIOD = 14

# OKX API基础地址
BASE_URL = "https://www.okx.com"

def okx_request(method, request_path, body=''):
    """发送OKX API请求"""
    timestamp = datetime.now(timezone.utc).isoformat(tol_seconds=True)
    if method == 'GET':
        message = timestamp + method + request_path
    else:
        message = timestamp + method + request_path + body
    
    mac = hmac.new(bytes(SECRET_KEY, 'utf-8'), bytes(message, 'utf-8'), hashlib.sha256)
    sign = base64.b64encode(mac.digest()).decode('utf-8')
    
    headers = {
        'OK-ACCESS-KEY': API_KEY,
        'OK-ACCESS-SIGN': sign,
        'OK-ACCESS-TIMESTAMP': timestamp,
        'OK-ACCESS-PASSPHRASE': PASSPHRASE,
        'Content-Type': 'application/json'
    }
    
    url = BASE_URL + request_path
    if method == 'GET':
        response = requests.get(url, headers=headers)
    else:
        response = requests.post(url, headers=headers, data=body)
    return response.json()

def get_klines(symbol='ETH-USDT', bar='5m', limit=50):
    """获取K线数据"""
    request_path = f"/api/v5/market/candles?instId={symbol}&bar={bar}&limit={limit}"
    result = okx_request('GET', request_path)
    if result.get('code') == '0':
        return result.get('data', [])
    return []

def calculate_rsi(klines):
    """计算RSI指标"""
    if len(klines) < RSI_PERIOD + 1:
        return 50
    
    closes = [float(k[4]) for k in klines]  # 收盘价
    gains, losses = [], []
    
    for i in range(1, len(closes)):
        change = closes[i] - closes[i-1]
        gains.append(change if change > 0 else 0)
        losses.append(-change if change < 0 else 0)
    
    avg_gain = sum(gains[-RSI_PERIOD:]) / RSI_PERIOD
    avg_loss = sum(losses[-RSI_PERIOD:]) / RSI_PERIOD
    
    if avg_loss == 0:
        return 100
    rs = avg_gain / avg_loss
    rsi = 100 - (100 / (1 + rs))
    return rsi

def get_account_balance():
    """获取账户余额"""
    result = okx_request('GET', '/api/v5/account/balance')
    if result.get('code') == '0':
        details = result.get('data', [{}])[0].get('details', [])
        for d in details:
            if d.get('ccy') == 'USDT':
                return float(d.get('availEq', 0))
    return 0

def send_wechat_message(title, content):
    """发送微信推送"""
    if not SERVERCHAN_KEY:
        return
    url = f"https://sctapi.ftqq.com/{SERVERCHAN_KEY}.send"
    data = {'title': title, 'desp': content}
    try:
        requests.post(url, data=data)
    except:
        pass

def main():
    """主函数：检查行情并发送信号"""
    if not all([API_KEY, SECRET_KEY, PASSPHRASE]):
        print("API密钥未配置")
        return
    
    # 1. 获取余额，检查是否已达5%亏损上限
    current_balance = get_account_balance()
    if current_balance > 0 and (TOTAL_CAPITAL - current_balance) >= MAX_LOSS_AMOUNT:
        send_wechat_message(
            "⚠️ 风控警报 - 交易暂停",
            f"累计亏损已达 {MAX_LOSS_AMOUNT:.4f} USDT（5%上限），暂停交易。当前余额：{current_balance:.4f} USDT"
        )
        return
    
    # 2. 获取K线并计算RSI
    klines = get_klines()
    if not klines:
        print("获取行情失败")
        return
    
    current_price = float(klines[0][4])
    rsi = calculate_rsi(klines)
    
    # 3. 判断交易信号
    signal = ""
    if rsi < RSI_OVERSOLD:
        signal = "🔔 **买入信号**"
        reason = f"RSI = {rsi:.2f}，低于超卖阈值（{RSI_OVERSOLD}），可能迎来反弹机会。"
        action = "建议：在OKX App买入 ETH/USDT，金额控制在 5 USDT 以内，止损设 10%"
    else:
        signal = "⏸️ **观望信号**"
        reason = f"当前RSI = {rsi:.2f}，未触发买入条件（需RSI < {RSI_OVERSOLD}）。"
        action = "暂无操作建议"
    
    # 4. 推送微信消息
    content = f"""
**当前价格**：{current_price:.4f} USDT
**RSI指标**：{rsi:.2f}
**账户余额**：{current_balance:.4f} USDT
**当日亏损上限**：{MAX_LOSS_AMOUNT:.4f} USDT

**信号判断**：{signal}

{reason}

{action}
    """
    
    send_wechat_message(f"ETH/USDT 交易信号 - {datetime.now().strftime('%Y-%m-%d %H:%M')}", content)
    print("信号已发送")

if __name__ == "__main__":
    main()
