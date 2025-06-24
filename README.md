# Midas-trading-bot
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
    <input type="number" id="valorTrade" placeholder="Ex.: 10 ou 0.1">
  </div>
</div>

<div id="trades" class="section">
  <h2>Histórico de Trades</h2>
  <div id="logTrades"></div>
</div>

<div class="modal" id="modalInfo">
  <h3>Sobre o Midas</h3>
  <p>O Midas Bot faz trades automáticos entre SOL e USDC na Solana, utilizando a API da Jupiter. Após ativar com uma taxa de 0.05 SOL, ele analisa o mercado, define automaticamente stop-loss, take-profit e executa operações reais de forma autônoma.</p>
  <button onclick="toggleModal()">Fechar</button>
</div>

<script src="https://unpkg.com/@solana/web3.js@1.74.2/lib/index.iife.js"></script>
<script>
const TAX_WALLET = "6Rsti4LKHo8oZynW9aJNEkLULuGsTVaj8kXqmsNGTKWp";
const TAX_AMOUNT_SOL = 0.05;
let publicKey = null;
let botAtivo = false;
let botInterval = null;
let ultimaEntrada = null;

const connection = new solanaWeb3.Connection("https://api.mainnet-beta.solana.com");

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

async function conectarWallet() {
  if (!window.solana || !window.solana.isPhantom) {
    alert("Phantom Wallet não encontrada.");
    return;
  }
  try {
    const resp = await window.solana.connect();
    publicKey = resp.publicKey.toString();
    document.getElementById("connectWallet").textContent = publicKey.slice(0, 4) + "..." + publicKey.slice(-4);
    logar("Wallet conectada: " + publicKey);
  } catch (e) {
    alert("Erro ao conectar: " + e.message);
  }
}

async function checkTaxPaid() {
  const signatures = await connection.getSignaturesForAddress(new solanaWeb3.PublicKey(TAX_WALLET), {limit: 100});
  for (const sig of signatures) {
    const tx = await connection.getTransaction(sig.signature, { commitment: "confirmed" });
    if (tx) {
      const keys = tx.transaction.message.accountKeys.map(k => k.toBase58());
      const fromIndex = keys.indexOf(publicKey);
      const toIndex = keys.indexOf(TAX_WALLET);
      if (fromIndex !== -1 && toIndex !== -1) {
        const lamportsSent = tx.meta.preBalances[fromIndex] - tx.meta.postBalances[fromIndex];
        if (lamportsSent >= solanaWeb3.LAMPORTS_PER_SOL * TAX_AMOUNT_SOL) {
          return true;
        }
      }
    }
  }
  return false;
}

async function payTax() {
  const transaction = new solanaWeb3.Transaction().add(
    solanaWeb3.SystemProgram.transfer({
      fromPubkey: new solanaWeb3.PublicKey(publicKey),
      toPubkey: new solanaWeb3.PublicKey(TAX_WALLET),
      lamports: solanaWeb3.LAMPORTS_PER_SOL * TAX_AMOUNT_SOL,
    })
  );
  transaction.feePayer = new solanaWeb3.PublicKey(publicKey);
  const { blockhash } = await connection.getLatestBlockhash();
  transaction.recentBlockhash = blockhash;

  const signed = await window.solana.signTransaction(transaction);
  const signature = await connection.sendRawTransaction(signed.serialize());
  await connection.confirmTransaction(signature);

  return signature;
}

async function toggleBot() {
  if (!publicKey) return alert("Conecte sua wallet primeiro!");

  if (!botAtivo) {
    const hasPaid = await checkTaxPaid();
    if (!hasPaid) {
      const confirmPay = confirm("Você precisa pagar 0.05 SOL para ativar o bot. Deseja pagar?");
      if (!confirmPay) return;
      try {
        await payTax();
        alert("Taxa paga com sucesso! Bot ativado.");
      } catch (e) {
        alert("Pagamento falhou ou foi cancelado.");
        return;
      }
    }
    document.getElementById("statusBot").innerText = "Ativado";
    document.getElementById("botToggle").innerText = "Desativar Modo Automático";
    botAtivo = true;
    iniciarBot();
  } else {
    botAtivo = false;
    clearInterval(botInterval);
    ultimaEntrada = null;
    document.getElementById("statusBot").innerText = "Desativado";
    document.getElementById("botToggle").innerText = "Ativar Modo Automático";
    logar("Bot desativado manualmente.");
  }
}

function iniciarBot() {
  const perfil = document.getElementById("perfil").value;
  botInterval = setInterval(() => botInteligente(perfil), 15000);
  logar(`Bot rodando no perfil ${perfil}`);
}

// Você pode deixar sua lógica do bot aqui igual estava antes (botInteligente, obterPreco, executarSwap, etc).

function logar(mensagem) {
  const logDiv = document.getElementById("logTrades");
  const el = document.createElement("div");
  el.innerHTML = mensagem;
  logDiv.prepend(el);
}

document.getElementById("connectWallet").onclick = conectarWallet;
</script>
</body>
</html>
