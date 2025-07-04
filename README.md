<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Midas Trading Bot</title>
  <style>
    * { box-sizing: border-box; }
    body { margin: 0; background: #0e0e0e; font-family: Arial, sans-serif; color: #ffd700; }
    header { background: #1a1a1a; display: flex; justify-content: space-between; align-items: center; padding: 16px 24px; border-bottom: 1px solid #ffd700; position: relative; }
    .logo { font-weight: bold; font-size: 24px; }
    .connect-btn { background: #ffd700; color: #0e0e0e; padding: 8px 18px; border: none; cursor: pointer; font-weight: bold; border-radius: 4px; transition: 0.2s; }
    .connect-btn:hover { background-color: #e6c200; }
    .tabs { display: flex; justify-content: space-around; background: #1a1a1a; border-bottom: 1px solid #ffd700; }
    .tab { padding: 14px 20px; cursor: pointer; transition: 0.2s; }
    .tab:hover { background-color: rgba(255, 215, 0, 0.1); }
    .tab.active { background: #ffd700; color: #000; font-weight: bold; }
    .section { display: none; padding: 24px; }
    .section.active { display: block; }
    .form-group { margin-bottom: 20px; }
    label { display: block; margin-bottom: 8px; font-size: 15px; }
    input, select { padding: 8px; width: 100%; border: 1px solid #ffd700; border-radius: 4px; background-color: #1a1a1a; color: #ffd700; outline: none; }
    input:focus, select:focus { border-color: #fff35c; }
    button { padding: 10px 16px; background: #ffd700; border: none; font-weight: bold; cursor: pointer; border-radius: 4px; transition: 0.2s; }
    button:hover { background-color: #e6c200; }
    .info-icon { position: absolute; top: 18px; right: 20px; font-size: 14px; background: #ffd700; color: #000; border-radius: 50%; width: 22px; height: 22px; text-align: center; line-height: 22px; cursor: pointer; font-weight: bold; }
    .modal { display: none; position: fixed; top: 20%; left: 50%; transform: translateX(-50%); background: #1a1a1a; color: #ffd700; padding: 24px; border: 2px solid #ffd700; border-radius: 8px; z-index: 10; max-width: 400px; }
    #logTrades { max-height: 400px; overflow-y: auto; border: 1px solid #ffd700; border-radius: 6px; padding: 12px; background-color: #1a1a1a; }
    #logTrades div { margin-bottom: 10px; border-bottom: 1px solid #ffd70033; padding-bottom: 6px; }
  </style>
</head>
<body>
  <header>
    <div class="logo">Midas Bot</div>
    <button id="connectWallet" class="connect-btn">Conectar Wallet</button>
    <div class="info-icon" onclick="toggleModal()">?</div>
  </header>

  <div class="tabs">
    <div class="tab active" onclick="showTab('menu')">Menu</div>
    <div class="tab" onclick="showTab('config')">Configurações</div>
    <div class="tab" onclick="showTab('trades')">Trades</div>
  </div>

  <div id="menu" class="section active">
    <h2>Status do Bot</h2>
    <p>Status: <span id="statusBot">Desativado</span></p>
    <button onclick="toggleBot()" id="botToggle">Ativar Modo Automático</button>
  </div>

  <div id="config" class="section">
    <h2>Configurações do Bot</h2>
    <div class="form-group">
      <label for="perfil">Perfil:</label>
      <select id="perfil">
        <option value="conservador">Conservador</option>
        <option value="moderado">Moderado</option>
        <option value="agressivo">Agressivo</option>
      </select>
    </div>
    <div class="form-group">
      <label for="ativoEm">Ativo para operar:</label>
      <select id="ativoEm">
        <option value="usdc">USDC</option>
        <option value="sol">SOL</option>
      </select>
    </div>
    <div class="form-group">
      <label for="valorTrade">Valor por Trade:</label>
      <input type="number" id="valorTrade" placeholder="Ex.: 0.1" value="0.1">
    </div>
  </div>

  <div id="trades" class="section">
    <h2>Histórico de Trades</h2>
    <div id="logTrades"></div>
    <p>Preço Atual: <span id="precoAtual">--</span> USDC</p>
    <p>Stop-Loss: <span id="stopLoss">--</span> USDC</p>
    <p>Take-Profit: <span id="takeProfit">--</span> USDC</p>
  </div>

  <div class="modal" id="modalInfo">
    <h3>Sobre o Midas</h3>
    <p>Veja o código completo com funcionalidades no painel lateral ou solicite o arquivo .zip para download.</p>
    <button onclick="toggleModal()">Fechar</button>
  </div>

<script src="https://unpkg.com/@solana/web3.js@1.74.2/lib/index.iife.js"></script>
<script src="https://unpkg.com/@jup-ag/api@6.0.44/dist/index.iife.js"></script>
<script>
let publicKey = null;
let botAtivo = false;
let botInterval = null;
let priceHistory = [];

const connection = new solanaWeb3.Connection("https://api.mainnet-beta.solana.com");
const jupiter = createJupiterApiClient({ basePath: "https://quote-api.jup.ag" });

function showTab(id) {
  document.querySelectorAll('.tab').forEach(t => t.classList.remove('active'));
  document.querySelectorAll('.section').forEach(s => s.classList.remove('active'));
  document.querySelector(`.tab[onclick*="${id}"]`).classList.add('active');
  document.getElementById(id).classList.add('active');
}

function toggleModal() {
  const modal = document.getElementById('modalInfo');
  modal.style.display = modal.style.display === 'block' ? 'none' : 'block';
}

function logar(msg) {
  const log = document.getElementById("logTrades");
  const entry = document.createElement("div");
  entry.innerText = `[${new Date().toLocaleTimeString()}] ${msg}`;
  log.appendChild(entry);
  log.scrollTop = log.scrollHeight;
}

async function conectarWallet() {
  if (!window.solana || !window.solana.isPhantom) return alert("Phantom Wallet não encontrada.");
  try {
    const resp = await window.solana.connect();
    publicKey = resp.publicKey.toString();
    document.getElementById("connectWallet").textContent = publicKey.slice(0, 4) + "..." + publicKey.slice(-4);
    logar("Wallet conectada: " + publicKey);
  } catch (e) {
    alert("Erro ao conectar: " + e.message);
  }
}

async function getRealPrice() {
  try {
    const res = await fetch("https://api.coingecko.com/api/v3/simple/price?ids=solana&vs_currencies=usd");
    const data = await res.json();
    return data.solana.usd;
  } catch (e) {
    logar("Erro ao obter preço real: " + e.message);
    return null;
  }
}

function calcularMedia(periodo) {
  const slice = priceHistory.slice(-periodo);
  const sum = slice.reduce((a, b) => a + b, 0);
  return sum / periodo;
}

function detectarCruz() {
  if (priceHistory.length < 50) return null;
  const sma50 = calcularMedia(50);
  const sma200 = calcularMedia(200);
  const prev50 = calcularMedia(50, -1);
  const prev200 = calcularMedia(200, -1);

  if (prev50 < prev200 && sma50 > sma200) return "ouro";
  if (prev50 > prev200 && sma50 < sma200) return "morte";
  return null;
}

function calcularStopTake(price) {
  const stopLoss = price * 0.97;
  const takeProfit = price * 1.03;
  document.getElementById("precoAtual").innerText = price.toFixed(2);
  document.getElementById("stopLoss").innerText = stopLoss.toFixed(2);
  document.getElementById("takeProfit").innerText = takeProfit.toFixed(2);
  return { stopLoss, takeProfit };
}

// (Restante do código com as funções botInteligente, executeTrade e toggleBot mantido igual)
</script>
</body>
</html>

