# Controle-de-Cortes-de-cabelo
<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Barber Control — Controle de Cortes</title>
<!-- Chart.js CDN -->
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<style>
  :root{--bg:#f4f6f8;--card:#fff;--accent:#0b5;--muted:#666}
  body{font-family:Inter,Arial,Helvetica,sans-serif;background:var(--bg);margin:0;padding:20px}
  .container{max-width:1100px;margin:0 auto}
  header{display:flex;align-items:center;justify-content:space-between;margin-bottom:18px}
  h1{font-size:20px;margin:0}
  .card{background:var(--card);padding:14px;border-radius:10px;box-shadow:0 6px 18px rgba(2,10,20,0.06);margin-bottom:12px}
  form .row{display:flex;gap:10px}
  input,select{padding:8px;border-radius:6px;border:1px solid #ddd;font-size:14px}
  button{background:#111;color:#fff;padding:8px 12px;border-radius:6px;border:0;cursor:pointer}
  table{width:100%;border-collapse:collapse;margin-top:10px}
  th,td{padding:8px;border-bottom:1px solid #eee;text-align:left}
  .small{font-size:13px;color:var(--muted)}
  .actions button{margin-right:6px}
  .pill{display:inline-block;padding:6px 8px;border-radius:999px;background:#eef;}
  .danger{background:#ffdddd;color:#a00;padding:6px 8px;border-radius:6px}
  .success{background:#ddffea;color:#070;padding:6px 8px;border-radius:6px}
  .toolbar{display:flex;gap:8px;flex-wrap:wrap}
  @media(max-width:800px){form .row{flex-direction:column}}
  /* --- NOVO VISUAL PROFISSIONAL --- */
  body{background:#e9edf3;}
  header h1{font-size:24px;font-weight:700;}
  .card{border-left:5px solid #0b5;}
  button{display:flex;align-items:center;gap:6px;font-weight:600;transition:.2s;}
  button:hover{opacity:.85;transform:scale(1.02);} 
  .actions button{margin-bottom:4px;}
  .icon{font-size:16px;}
</style>
</head>
<body>
<div class="container">
  <header>
    <h1>Barber Control — Hyago</h1>
    <div class="small">Gerencie pagamentos, cortes e envie cobranças</div>
  </header>

  <div class="card">
    <h3>Cadastro de Cliente</h3>
    <form id="formCliente" onsubmit="event.preventDefault();addCliente()">
      <div class="row">
        <input id="nome" placeholder="Nome do cliente" required />
        <input id="telefone" placeholder="Telefone (WhatsApp) - opcional" />
        <input id="cortesMes" type="number" min="1" value="2" style="width:110px" />
        <input id="valor" type="number" min="0" value="80" style="width:120px" />
        <button type="submit">Adicionar</button>
      </div>
      <div class="small">Caso queira enviar cobrança pelo WhatsApp, preencha o telefone no formato internacional (ex: 5511999998888)</div>
    </form>
  </div>

  <div class="card">
    <div style="display:flex;justify-content:space-between;align-items:center">
      <h3>Clientes</h3>
      <div class="toolbar">
        <button onclick="baixarCSV()">Exportar CSV</button>
        <button onclick="gerarRelatorio()">Gerar relatório (CSV)</button>
        <button onclick="fecharMes()">Fechar mês (salvar histórico e resetar)</button>
        <button onclick="receberMes()">Receber mês (marcar pagos)</button>
      </div>
    </div>

    <div id="tabelaWrap"></div>
  </div>

  <div class="card">
    <h3>Dashboard</h3>
    <canvas id="chartCuts" height="120"></canvas>
    <div style="display:flex;gap:12px;margin-top:10px;align-items:center">
      <div class="pill" id="totais"></div>
      <div class="small">Gráfico mostra total de cortes feitos vs restantes (todos os clientes)</div>
    </div>
  </div>

  <div class="card">
    <h3>Histórico por Mês</h3>
    <div style="display:flex;gap:8px;align-items:center">
      <select id="selMes" onchange="mostrarHistorico()"></select>
      <button onclick="downloadHistoricoCSV()">Baixar histórico (CSV)</button>
      <button onclick="limparHistorico()" class="danger">Limpar histórico</button>
    </div>
    <div id="histArea" style="margin-top:10px"></div>
  </div>

  <footer style="margin-top:20px;text-align:center;color:var(--muted);font-size:13px">Salvo no armazenamento local do navegador. Faça backup exportando CSV.</footer>
</div>

<script>
// LocalStorage keys
const KEY = 'barber_control_clients_v1';
const KEY_HISTORY = 'barber_control_history_v1';

let clients = JSON.parse(localStorage.getItem(KEY)) || [];
let history = JSON.parse(localStorage.getItem(KEY_HISTORY)) || {};

function salvar(){ localStorage.setItem(KEY, JSON.stringify(clients)); }
function salvarHist(){ localStorage.setItem(KEY_HISTORY, JSON.stringify(history)); }

function formatCurrency(v){ return 'R$ ' + Number(v).toFixed(2).replace('.',','); }

function addCliente(){
  const nome = document.getElementById('nome').value.trim();
  const telefone = document.getElementById('telefone').value.trim();
  const cortesMes = Number(document.getElementById('cortesMes').value) || 2;
  const valor = Number(document.getElementById('valor').value) || 80;
  if(!nome) return alert('Digite o nome');
  clients.push({ nome, telefone, cortesFeitos:0, cortesMes, valor, pago:false });
  salvar();
  document.getElementById('formCliente').reset();
  renderTable();
  atualizarChart();
}

function editarCliente(i){
  const c = clients[i];
  const novo = prompt('Editar nome', c.nome);
  if(novo===null) return;
  clients[i].nome = novo.trim()||c.nome;
  salvar(); renderTable(); atualizarChart();
}

function excluirCliente(i){ if(!confirm('Excluir cliente?')) return; clients.splice(i,1); salvar(); renderTable(); atualizarChart(); }

function registrarCorte(i){
  if(clients[i].cortesFeitos >= clients[i].cortesMes){ alert('Já usou todos os cortes do mês'); return; }
  clients[i].cortesFeitos++;
  salvar(); renderTable(); atualizarChart();
}

function togglePago(i){ clients[i].pago = !clients[i].pago; salvar(); renderTable(); }

function receberMes(){ if(!confirm('Marcar todos os clientes como PAGOS?')) return; clients = clients.map(c=>({...c, pago:true})); salvar(); renderTable(); }

function fecharMes(){
  if(!confirm('Fechar mês: salvar snapshot no histórico e resetar cortes/pagamentos?')) return;
  const key = new Date().toISOString().slice(0,7); // YYYY-MM
  const snapshot = clients.map(c=>({ nome:c.nome, cortesFeitos:c.cortesFeitos, cortesMes:c.cortesMes, valor:c.valor, pago:c.pago }));
  history[key] = { date: key, snapshot, totals: calcTotals() };
  salvarHist();
  // reset
  clients = clients.map(c=>({ ...c, cortesFeitos:0, pago:false }));
  salvar(); renderTable(); atualizarChart(); carregarMeses(); alert('Mês fechado e salvo: '+key);
}

function calcTotals(){
  let totalRecebido=0, totalPendente=0, cortesFeitos=0, cortesRest=0;
  for(const c of clients){ cuts = c.cortesFeitos; cortesFeitos += cuts; cortesRest += (c.cortesMes - cuts);
    if(c.pago) totalRecebido += c.valor; else totalPendente += c.valor;
  }
  return { totalRecebido, totalPendente, cortesFeitos, cortesRest };
}

function renderTable(){
  if(clients.length===0){ document.getElementById('tabelaWrap').innerHTML = '<div class="small">Nenhum cliente cadastrado.</div>'; return; }
  let html = '<table><thead><tr><th>Nome</th><th>Cortes (Feitos / Mês)</th><th>Restam</th><th>Valor</th><th>Status</th><th>Ações</th></tr></thead><tbody>';
  clients.forEach((c,i)=>{
    const rest = c.cortesMes - c.cortesFeitos;
    html += `<tr>
      <td><b>${c.nome}</b><div class="small">${c.telefone||''}</div></td>
      <td>${c.cortesFeitos} / ${c.cortesMes}</td>
      <td>${rest}</td>
      <td>${formatCurrency(c.valor)}</td>
      <td>${c.pago?'<span class="success">Pago</span>':'<span class="danger">Pendente</span>'}</td>
      <td class="actions">
        <button onclick="registrarCorte(${i})">Registrar Corte</button>
        <button onclick="togglePago(${i})">Marcar/Desmarcar Pago</button>
        <button onclick="editarCliente(${i})">Editar</button>
        <button onclick="excluirCliente(${i})">Excluir</button>
        <button onclick="enviarCobranca(${i})">Enviar Cobrança</button>
      </td>
    </tr>`;
  });
  html += '</tbody></table>';
  document.getElementById('tabelaWrap').innerHTML = html;
  const t = calcTotals();
  document.getElementById('totais').innerText = `Recebido: ${formatCurrency(t.totalRecebido)} • Pendente: ${formatCurrency(t.totalPendente)} • Cortes feitos: ${t.cortesFeitos} • Restantes: ${t.cortesRest}`;
}

function enviarCobranca(i){
  const c = clients[i];
  const chavePIX = 'hyagosousasous@gmail.com';
  const texto = `Olá ${c.nome},%0A%0A`+
    `Segue o lembrete do pagamento do mês:%0A`+
    `💈 Valor: R$ ${c.valor.toFixed(2).replace('.',',')}%0A`+
    `💳 PIX: ${chavePIX}%0A%0A`+
    `Obrigado!`;

  if(c.telefone){
    const phone = c.telefone.replace(/[^0-9]/g,'');
    const url = `https://web.whatsapp.com/send?phone=${phone}&text=${texto}`;
    window.open(url,'_blank');
  } else {
    alert('Este cliente não tem telefone cadastrado para WhatsApp.');
  }
}(i){
  const c = clients[i];
  const chavePIX = 'hyagosousasous@gmail.com';
  const texto = `Olá ${c.nome},\nLembrete: valor do mês R$ ${c.valor.toFixed(2).replace('.',',')} para os cortes. Por favor enviar via PIX para a chave: ${chavePIX}. Obrigado!`;
  // abrir WhatsApp se tiver telefone, senão abrir mailto
  if(c.telefone){
    const phone = c.telefone.replace(/[^0-9]/g,'');
    const url = `https://wa.me/${phone}?text=` + encodeURIComponent(texto);
    window.open(url,'_blank');
  } else {
    // abrir e-mail com corpo
    const mailto = `mailto:?subject=${encodeURIComponent('Cobrança de cortes')}&body=${encodeURIComponent(texto)}`;
    window.open(mailto,'_blank');
  }
}

// CSV export
function baixarCSV(){
  if(clients.length===0){ alert('Sem dados para exportar'); return; }
  const rows = [['Nome','Telefone','CortesFeitos','CortesMes','Valor','Pago']];
  clients.forEach(c=> rows.push([c.nome,c.telefone,c.cortesFeitos,c.cortesMes,c.valor,c.pago]));
  const csv = rows.map(r=>r.map(v=>`"${String(v).replace(/"/g,'""')}"`).join(',')).join('\n');
  const blob = new Blob([csv],{type:'text/csv;charset=utf-8;'});
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a'); a.href = url; a.download = 'barber_clients.csv'; a.click(); URL.revokeObjectURL(url);
}

function gerarRelatorio(){ baixarCSV(); }

// History functions
function carregarMeses(){
  const sel = document.getElementById('selMes'); sel.innerHTML = '<option value="">Selecionar mês</option>';
  const keys = Object.keys(history).sort().reverse();
  for(const k of keys){ const opt = document.createElement('option'); opt.value=k; opt.textContent=k; sel.appendChild(opt); }
}

function mostrarHistorico(){
  const key = document.getElementById('selMes').value; const area = document.getElementById('histArea'); area.innerHTML='';
  if(!key) return;
  const h = history[key];
  if(!h) return area.innerText='Histórico não encontrado.';
  let html = `<div><b>Mês:</b> ${key} • Totais: Recebido ${formatCurrency(h.totals.totalRecebido)} • Pendente ${formatCurrency(h.totals.totalPendente)} • Cortes feitos ${h.totals.cortesFeitos}</div>`;
  html += '<table><thead><tr><th>Nome</th><th>CortesFeitos</th><th>CortesMes</th><th>Valor</th><th>Pago</th></tr></thead><tbody>';
  h.snapshot.forEach(r=> html += `<tr><td>${r.nome}</td><td>${r.cortesFeitos}</td><td>${r.cortesMes}</td><td>${formatCurrency(r.valor)}</td><td>${r.pago}</td></tr>`);
  html += '</tbody></table>';
  area.innerHTML = html;
}

function downloadHistoricoCSV(){
  const key = document.getElementById('selMes').value; if(!key){ alert('Selecione um mês'); return; }
  const h = history[key]; if(!h){ alert('Histórico não encontrado'); return; }
  const rows = [['Nome','CortesFeitos','CortesMes','Valor','Pago']];
  h.snapshot.forEach(r=> rows.push([r.nome,r.cortesFeitos,r.cortesMes,r.valor,r.pago]));
  const csv = rows.map(r=>r.map(v=>`"${String(v).replace(/"/g,'""')}"`).join(',')).join('\n');
  const blob = new Blob([csv],{type:'text/csv;charset=utf-8;'});
  const url = URL.createObjectURL(blob); const a=document.createElement('a'); a.href=url; a.download = `barber_history_${key}.csv`; a.click(); URL.revokeObjectURL(url);
}

function limparHistorico(){ if(!confirm('Limpar todo o histórico?')) return; history={}; salvarHist(); carregarMeses(); document.getElementById('histArea').innerHTML=''; alert('Histórico limpo'); }

// Chart
let chart=null;
function atualizarChart(){
  const labels = clients.map(c=>c.nome);
  const feitos = clients.map(c=>c.cortesFeitos);
  const restantes = clients.map(c=>Math.max(0,c.cortesMes - c.cortesFeitos));
  const ctx = document.getElementById('chartCuts').getContext('2d');
  const data = { labels, datasets: [ { label:'Feitos', data:feitos }, { label:'Restantes', data:restantes } ] };
  if(chart) chart.data = data, chart.update(); else chart = new Chart(ctx, { type:'bar', data, options:{ responsive:true, scales:{ y:{ beginAtZero:true, precision:0 } } } });
}

function init(){ renderTable(); atualizarChart(); carregarMeses(); }
init();
</script>
</body>
</html>
