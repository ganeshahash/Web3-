# Web3-from flask import Flask, jsonify, render_template_string, request, session
from flask_cors import CORS
import ccxt
import pandas as pd
import pandas_ta as ta
import os

app = Flask(__name__)
CORS(app)
app.secret_key = os.urandom(24)

# Usuários simulados
users = {
    "admin": "1234",
    "usuario": "senha"
}

# HTML da interface
HTML_TEMPLATE = '''
<!DOCTYPE html>
<html>
<head>
  <title>Cripto IA</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <script src="https://cdn.jsdelivr.net/npm/axios/dist/axios.min.js"></script>
</head>
<body>
  <h2>Cripto Análise IA</h2>
  <div id="login">
    <input type="text" id="user" placeholder="Usuário"><br>
    <input type="password" id="pass" placeholder="Senha"><br>
    <button onclick="login()">Entrar</button>
    <p id="login-msg"></p>
  </div>
  <div id="app" style="display:none;">
    <p>Bem-vindo! <button onclick="logout()">Sair</button></p>
    <select id="symbol">
      <option>BTC/USDT</option>
      <option>ETH/USDT</option>
      <option>SOL/USDT</option>
      <option>BNB/USDT</option>
    </select>
    <select id="timeframe">
      <option value="1h">1H</option>
      <option value="4h">4H</option>
      <option value="1d">1D</option>
    </select>
    <button onclick="analisar()">Analisar</button>
    <pre id="result"></pre>
    <div id="tvwidget">
      <script src="https://s3.tradingview.com/tv.js"></script>
      <script>
        new TradingView.widget({
          "width": "100%",
          "height": 500,
          "symbol": "BINANCE:BTCUSDT",
          "interval": "60",
          "theme": "dark",
          "container_id": "tvwidget"
        });
      </script>
    </div>
  </div>
  <script>
    function login() {
      axios.post('/login', {
        username: document.getElementById('user').value,
        password: document.getElementById('pass').value
      }).then(res => {
        document.getElementById('login').style.display = 'none';
        document.getElementById('app').style.display = 'block';
      }).catch(() => {
        document.getElementById('login-msg').innerText = 'Falha no login';
      });
    }

    function logout() {
      axios.post('/logout').then(() => location.reload());
    }

    function analisar() {
      axios.post('/analisar', {
        symbol: document.getElementById('symbol').value,
        timeframe: document.getElementById('timeframe').value
      }).then(res => {
        document.getElementById('result').innerText = JSON.stringify(res.data, null, 2);
      }).catch(() => {
        alert('Erro na análise');
      });
    }
  </script>
</body>
</html>
'''

@app.route('/')
def home():
    return render_template_string(HTML_TEMPLATE)

@app.route('/login', methods=['POST'])
def login():
    data = request.json
    if users.get(data.get('username')) == data.get('password'):
        session['user'] = data['username']
        return jsonify({"message": "Login bem-sucedido"})
    return jsonify({"message": "Credenciais inválidas"}), 401

@app.route('/logout', methods=['POST'])
def logout():
    session.pop('user', None)
    return jsonify({"message": "Logout feito"})

@app.route('/analisar', methods=['POST'])
def analisar():
    if 'user' not in session:
        return jsonify({"message": "Não autenticado"}), 401

    data = request.json
    symbol = data.get('symbol', 'BTC/USDT')
    timeframe = data.get('timeframe', '1h')

    try:
        exchange = ccxt.binance()
        candles = exchange.fetch_ohlcv(symbol, timeframe=timeframe, limit=100)
        df = pd.DataFrame(candles, columns=['timestamp', 'open', 'high', 'low', 'close', 'volume'])

        df['rsi'] = ta.rsi(df['close'], length=14)
        macd = ta.macd(df['close'])
        df['macd'] = macd['MACD_12_26_9']
        df['signal'] = macd['MACDs_12_26_9']
        df['ema_9'] = ta.ema(df['close'], length=9)
        df['ema_20'] = ta.ema(df['close'], length=20)

        close = df['close'].iloc[-1]
        rsi = df['rsi'].iloc[-1]
        macd = df['macd'].iloc[-1]
        signal = df['signal'].iloc[-1]
        ema9 = df['ema_9'].iloc[-1]
        ema20 = df['ema_20'].iloc[-1]

        recomendacao = "neutro"
        if rsi < 30 and macd > signal and close > ema9 > ema20:
            recomendacao = "compra"
        elif rsi > 70 and macd < signal and close < ema9:
            recomendacao = "venda"

        return jsonify({
            "close": close,
            "rsi": rsi,
            "macd": macd,
            "signal": signal,
            "ema_9": ema9,
            "ema_20": ema20,
            "recomendacao": recomendacao
        })
    except Exception as e:
        return jsonify({"erro": str(e)}), 500

if __name__ == '__main__':
    app.run()
