const appId = '72324';
const redirectUri = 'http://localhost:3000/';
let ws;
let token;
let masterWs;

const loginBtn = document.getElementById('login-btn');
const userInfo = document.getElementById('user-info');
const userName = document.getElementById('user-name');
const contractForm = document.getElementById('contract-form');
const symbolDropdown = document.getElementById('symbol-dropdown');
const buyBtn = document.getElementById('buy-contract');
const result = document.getElementById('buy-result');
const copyTradingSection = document.getElementById('copy-trading');
const masterTokenInput = document.getElementById('master-token');
const copyBtn = document.getElementById('start-copy');
const copyStatus = document.getElementById('copy-status');

loginBtn.onclick = () => {
  const url = `https://oauth.deriv.com/oauth2/authorize?app_id=${appId}&redirect_uri=${encodeURIComponent(redirectUri)}`;
  window.location.href = url;
};

const params = new URLSearchParams(window.location.search);
token = params.get('token1');

if (token) {
  ws = new WebSocket('wss://ws.derivws.com/websockets/v3');

  ws.onopen = () => {
    ws.send(JSON.stringify({ authorize: token }));
  };

  ws.onmessage = (msg) => {
    const data = JSON.parse(msg.data);

    if (data.msg_type === 'authorize') {
      userName.textContent = data.authorize.full_name || data.authorize.loginid;
      userInfo.classList.remove('hidden');
      contractForm.classList.remove('hidden');
      copyTradingSection.classList.remove('hidden');
      loadSymbols();
    }

    if (data.msg_type === 'buy') {
      result.textContent = `✅ Contract bought. ID: ${data.buy.contract_id}`;
    }

    if (data.msg_type === 'active_symbols') {
      symbolDropdown.innerHTML = '';
      data.active_symbols.forEach(symbol => {
        const option = document.createElement('option');
        option.value = symbol.symbol;
        option.textContent = `${symbol.display_name} (${symbol.symbol})`;
        symbolDropdown.appendChild(option);
      });
    }

    if (data.error) {
      result.textContent = `❌ Error: ${data.error.message}`;
    }
  };
}

function loadSymbols() {
  ws.send(JSON.stringify({ active_symbols: 'brief', product_type: 'basic' }));
}

buyBtn.onclick = () => {
  const type = document.getElementById('contract-type').value;
  const amount = document.getElementById('stake').value;
  const symbol = symbolDropdown.value;

  const buyRequest = {
    buy: 1,
    price: amount,
    parameters: {
      amount: amount,
      basis: 'stake',
      contract_type: type,
      currency: 'USD',
      duration: 1,
      duration_unit: 't',
      symbol: symbol
    }
  };

  ws.send(JSON.stringify(buyRequest));
};

copyBtn.onclick = () => {
  const masterToken = masterTokenInput.value.trim();
  if (!masterToken) return;

  masterWs = new WebSocket('wss://ws.derivws.com/websockets/v3');
  masterWs.onopen = () => {
    masterWs.send(JSON.stringify({ authorize: masterToken }));
    copyStatus.textContent = '🔄 Listening for trades...';
  };

  masterWs.onmessage = (msg) => {
    const data = JSON.parse(msg.data);

    if (data.msg_type === 'authorize') {
      masterWs.send(JSON.stringify({ transaction: 1, subscribe: 1 }));
    }

    if (data.msg_type === 'transaction' && data.transaction.action === 'buy') {
      const masterTrade = data.transaction;

      const mirroredBuy = {
        buy: 1,
        price: masterTrade.amount,
        parameters: {
          amount: masterTrade.amount,
          basis: 'stake',
          contract_type: masterTrade.contract_type,
          currency: masterTrade.currency,
          duration: 1,
          duration_unit: 't',
          symbol: masterTrade.symbol
        }
      };

      ws.send(JSON.stringify(mirroredBuy));
      copyStatus.textContent = `✅ Mirrored buy: ${masterTrade.contract_type} on ${masterTrade.symbol}`;
    }
  };
};
