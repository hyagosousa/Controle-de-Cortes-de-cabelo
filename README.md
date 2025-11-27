<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Barber Control — Controle de Cortes</title>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<style>
  :root{--bg:#1e1f25;--card:#2a2c33;--accent:#4caf50;--muted:#9aa0a6}
  body{margin:0;font-family:Inter, system-ui, -apple-system, 'Segoe UI', Roboto, Arial;background:var(--bg);color:#eef}
  .container{max-width:1100px;margin:18px auto;padding:12px}
  header{text-align:center;margin-bottom:16px}
  header h1{color:var(--accent);margin:0;font-size:28px}
  .card{background:var(--card);border-radius:12px;padding:16px;margin-bottom:14px;box-shadow:0 6px 18px rgba(0,0,0,0.5)}
  form .row{display:flex;gap:10px;align-items:center}
  input,select{padding:10px;border-radius:8px;border:1px solid #3a3b41;background:#161619;color:#fff}
  input::placeholder{color:#888}
  button{background:var(--accent);color:#fff;border:0;padding:10px 14px;border-radius:8px;cursor:pointer;font-weight:600}
  button.ghost{background:transparent;border:1px solid #3a3b41}
  table{width:100%;border-collapse:collapse;margin-top:12px}
  th,td{padding:10px;border-bottom:1px solid #34343a;text-align:left}
  th{color:var(--accent);background:#26272b}
  .small{color:var(--muted);font-size:13px}
  .pill{display:inline-block;padding:6px 10px;border-radius:999px;background:#223a2b;color:#cfeed8;font-weight:600}
  .danger{background:#3a1f1f;color:#ffb3b3;padding:6px 10px;border-radius:8px}
  .success{background:#123214;color:#bff0c9;padding:6px 10px;border-radius:8px}
  .actions button{margin-right:6px;margin-bottom:6px}
  @media(max-width:900px){form .row{flex-direction:column} .container{padding:8px}}
</style>
</head>
<body>
<div class="container">
  <header>
    <h1>Barber Control — Hyago</h1>
    <div class="small">Controle clientes, pagamentos, cortes e cobranças (PIX)</div>
  </header>

  <div class="card">
    <h3>Cadastro de Cliente</h3>
    <form id="formCliente" onsubmit="event.preventDefault();addCliente()">
      <div class="row">
        <input id="nome" placeholder="Nome do cliente" required />
        <input id="telefone" placeholder="Telefone (WhatsApp) - opcional" />
        <input id="cortesMes" type="number" min="1" value="2" style="width:120px" placeholder="Cortes/mês" />
        <input id="valor" type="number" min="0" step="0.01" value="80" style="width:140px" placeholder="Valor do plano (R$)" />
        <button type="submit">Adicionar</button>
      </div>
      <div class="small" style="margin-top:8px">Telefone: formato internacional ex: 5511999998888 — Valor em reais (ex: 80.00)</div>
    </form>
  </div>

  <div class="card">
    <div style="display:flex;justify-content:space-between;align-items:center">
      <h3 style="margin:0">Clientes</h3>
      <div style="display:flex;gap:8px">
        <button onclick="baixarCSV()" class="ghost">Exportar CSV</button>
        <button onclick="gerarRelatorio()" class="ghost">Relatório</button>
        <button onclick="fecharMes()" class="ghost">Fechar mês</button>
        <button onclick="receberMes()" class="ghost">Receber mês</button>
      </div>
    </div>

    <div id="tabelaWrap"></div>
  </div>

  <div class="card">
    <h3>Dashboard</h3>
    <canvas id="chartCuts" height="120"></canvas>
    <div style="display:flex;gap:12px;margin-top:12px;align-items:center">
      <div class="pill" id="totais"></div>
      <div class="small">Gráfico: cortes feitos x restantes</div>
    </div>
  </div>

  <div class="card">
    <h3>Histórico Mensal</h3>
    <div style="display:flex;gap:8px;align-items:center;margin-bottom:10px">
      <select id="selMes"></select>
      <button onclick="mostrarHistorico()">Abrir mês</button>
      <button onclick="downloadHistoricoCSV()">Baixar CSV mês</button>
      <button onclick="limparHistorico()" class="danger">Limpar histórico</button>
    </div>
    <div id="histArea" class="small"></div>
  </div>

  <footer style="text-align:center;margin-top:14px;color:var(--muted)">Salvo localmente no navegador. Faça backup exportando CSV.</footer>
</div>

<script>
// Keys
const KEY = 'barber_control_clients_v2';
const KEY_HISTORY = 'barber_control_history_v2';

// State
let clients = JSON.parse(localStorage.getItem(KEY)) || [];
let history = JSON.parse(localStorage.getItem(KEY_HISTORY)) || {}; // { '2025-11': { date:'2025-11', snapshot:[...], totals:{}} }

// Helpers
function salvar(){ localStorage.setItem(KEY, JSON.stringify(clients)); }
function salvarHist(){ localStorage.setItem(KEY_HISTORY, JSON.stringify(history)); }
function formatCurrency(v){ return 'R$ ' + Number(v).toFixed(2).replace('.',','); }

// Add client
function addCliente(){
  const nome = document.getElementById('nome').value.trim();
  const telefone = document.getElementById('telefone').value.trim();
  const cortesMes = Number(document.getElementById('cortesMes').value) || 2;
  const valor = Number(document.getElementById('valor').value) || 0;
  if(!nome) return alert('Digite o nome do cliente');

  clients.push({ nome, telefone, cortesFeitos:0, cortesMes, valor, pago:false });
  salvar();
  document.getElementById('formCliente').reset();
  renderTable(); atualizarChart();
}

// Edit
function editarCliente(i){
  const c = clients[i];
  const novo = prompt('Editar nome do cliente', c.nome);
  if(novo === null) return;
  clients[i].nome = novo.trim() || c.nome;
  salvar(); renderTable(); atualizarChart();
}

function excluirCliente(i){ if(!confirm('Excluir cliente?')) return; clients.splice(i,1); salvar(); renderTable(); atualizarChart(); }

// Registrar corte — principal ação ao cortar
function registrarCorte(i){
  const c = clients[i];
  if(c.cortesFeitos >= c.cortesMes){ alert('Este cliente já usou todos os cortes do mês — renovar plano.'); return; }

  // se antes do corte restava 2 e agora ficará 1, enviamos sugestão de agendamento
  const restAntes = c.cortesMes - c.cortesFeitos;

  c.cortesFeitos += 1;
  salvar(); renderTable(); atualizarChart();

  const restDepois = c.cortesMes - c.cortesFeitos;
  if(restDepois === 1){ // avisar via WhatsApp com pré-visualização
    if(c.telefone){
      const msg = `Olá ${c.nome},%0A%0AFalta apenas 1 corte para usar todo seu plano.%0AQuer agendar?`;
      const phone = c.telefone.replace(/[^0-9]/g,'');
      const url = `https://web.whatsapp.com/send?phone=${phone}&text=${msg}`;
      window.open(url,'_blank');
    }
  }

  if(c.cortesFeitos >= c.cortesMes){
    alert('Renove o plano');
  }
}

// marcar pagamento
function togglePago(i){ clients[i].pago = !clients[i].pago; salvar(); renderTable(); }

// renovar plano (zera cortes e marca pendente)
function renovarPlano(i){ if(!confirm('Renovar plano para '+clients[i].nome+' ?')) return; clients[i].cortesFeitos = 0; clients[i].pago = false; salvar(); renderTable(); atualizarChart(); }

// cobrar via WhatsApp (pré-visualização)
function enviarCobranca(i){
  const c = clients[i];
  if(!c) return;
  if(!c.telefone){ alert('Cliente não possui telefone cadastrado'); return; }
  const chavePIX = 'hyagosousasous@gmail.com';
  const texto = `Olá ${c.nome},%0A%0ASegue o lembrete do pagamento do mês:%0A💈 Valor: ${formatCurrency(c.valor)}%0A💳 PIX: ${chavePIX}%0A%0AObrigado!`;
  const phone = c.telefone.replace(/[^0-9]/g,'');
  const url = `https://web.whatsapp.com/send?phone=${phone}&text=${texto}`;
  window.open(url,'_blank');
}

// Receber mês — marca todos como pagos
function receberMes(){ if(!confirm('Marcar todos os clientes como PAGOS?')) return; clients = clients.map(c=>({...c,pago:true})); salvar(); renderTable(); }

// Fechar mês — salva snapshot no histórico e reseta cortes/pagamentos
function fecharMes(){
  if(!confirm('Fechar mês: salvar histórico e resetar cortes/pagamentos?')) return;
  const key = new Date().toISOString().slice(0,7); // YYYY-MM
  const snapshot = clients.map(c=>({ nome:c.nome, cortesFeitos:c.cortesFeitos, cortesMes:c.cortesMes, valor:c.valor, pago:c.pago }));
  history[key] = { date:key, snapshot, totals: calcTotals(clients) };
  salvarHist();
  // reset clientes
  clients = clients.map(c=>({...c, cortesFeitos:0, pago:false}));
  salvar(); renderTable(); atualizarChart(); carregarMeses(); alert('Mês fechado e salvo: '+key);
}

function calcTotals(list){
  let totalRecebido=0, totalPendente=0, cortesFeitos=0, cortesRest=0;
  for(const c of list){ const cuts=c.cortesFeito||c.cortesFeitos; cortesFeito = cuts; cortesFeitos += cuts; cortesRest += (c.cortesMes - cuts); if(c.pago) totalRecebido += c.valor; else totalPendente += c.valor; }
  return { totalRecebido, totalPendente, cortesFeitos, cortesRest };
}

// Export CSV
function baixarCSV(){
  if(clients.length===0){ alert('Sem dados para exportar'); return; }
  const rows = [['Nome','Telefone','CortesFeitos','CortesMes','Valor','Pago']];
  clients.forEach(c=> rows.push([c.nome,c.telefone,c.cortesFeitos,c.cortesMes,c.valor,c.pago]));
  const csv = rows.map(r=>r.map(v=>`"${String(v).replace(/"/g,'""')}"`).join(',')).join('\n');
  const blob = new Blob([csv],{type:'text/csv;charset=utf-8;'});
  const url = URL.createObjectURL(blob); const a=document.createElement('a'); a.href=url; a.download='barber_clients.csv'; a.click(); URL.revokeObjectURL(url);
}
function gerarRelatorio(){ baixarCSV(); }

// Histórico UI
function carregarMeses(){ const sel=document.getElementById('selMes'); sel.innerHTML=''; const keys=Object.keys(history).sort().reverse(); const opt=document.createElement('option'); opt.value=''; opt.textContent='Selecionar mês'; sel.appendChild(opt); for(const k of keys){ const o=document.createElement('option'); o.value=k; o.textContent=k; sel.appendChild(o); } }

function mostrarHistorico(){ const key=document.getElementById('selMes').value; const area=document.getElementById('histArea'); area.innerHTML=''; if(!key) return alert('Selecione um mês'); const h=history[key]; if(!h) return area.innerText='Histórico não encontrado'; let html=`<div style="margin-bottom:8px"><b>Mês:</b> ${key}</div>`; html += '<table><thead><tr><th>Nome</th><th>CortesFeitos</th><th>CortesMes</th><th>Valor</th><th>Pago</th></tr></thead><tbody>'; h.snapshot.forEach(r=> html+=`<tr><td>${r.nome}</td><td>${r.cortesFeitos}</td><td>${r.cortesMes}</td><td>${formatCurrency(r.valor)}</td><td>${r.pago}</td></tr>`); html += '</tbody></table>'; area.innerHTML=html; }

function downloadHistoricoCSV(){ const key=document.getElementById('selMes').value; if(!key) return alert('Selecione um mês'); const h=history[key]; if(!h) return alert('Histórico não encontrado'); const rows=[['Nome','CortesFeitos','CortesMes','Valor','Pago']]; h.snapshot.forEach(r=> rows.push([r.nome,r.cortesFeitos,r.cortesMes,r.valor,r.pago])); const csv=rows.map(r=>r.map(v=>`"${String(v).replace(/"/g,'""')}"`).join(',')).join('\n'); const blob=new Blob([csv],{type:'text/csv;charset=utf-8;'}); const url=URL.createObjectURL(blob); const a=document.createElement('a'); a.href=url; a.download=`barber_history_${key}.csv`; a.click(); URL.revokeObjectURL(url); }

function limparHistorico(){ if(!confirm('Apagar todo o histórico?')) return; history={}; salvarHist(); carregarMeses(); document.getElementById('histArea').innerHTML=''; alert('Histórico limpo'); }

// Chart
let chart=null;
function atualizarChart(){
  const labels = clients.map(c=>c.nome);
  const feitos = clients.map(c=>c.cortesFeitos);
  const restantes = clients.map(c=>Math.max(0,c.cortesMes - c.cortesFeitos));
  const ctx = document.getElementById('chartCuts').getContext('2d');
  const data = { labels, datasets:[ { label:'Feitos', data:feitos }, { label:'Restantes', data:restantes } ] };
  if(chart){ chart.data = data; chart.update(); } else { chart = new Chart(ctx, { type:'bar', data, options:{ responsive:true, scales:{ y:{ beginAtZero:true, precision:0 } } } }); }
}

// Render table
function renderTable(){
  const wrap = document.getElementById('tabelaWrap');
  if(clients.length===0){ wrap.innerHTML = '<div class="small">Nenhum cliente cadastrado.</div>'; document.getElementById('totais').innerText='Recebido: R$ 0,00 • Pendente: R$ 0,00 • Cortes feitos: 0 • Restantes: 0'; return; }
  let html = '<table><thead><tr><th>Nome</th><th>Cortes (Feitos / Mês)</th><th>Restam</th><th>Valor</th><th>Status</th><th>Ações</th></tr></thead><tbody>';
  clients.forEach((c,i)=>{
    const rest = c.cortesMes - c.cortesFeitos;
    html += `<tr><td><b>${c.nome}</b><div class="small">${c.telefone||''}</div></td><td>${c.cortesFeitos} / ${c.cortesMes}</td><td>${rest}</td><td>${formatCurrency(c.valor)}</td><td>${c.pago?'<span class="success">Pago</span>':'<span class="danger">Pendente</span>'}</td><td class="actions">`;
    html += `<button onclick="registrarCorte(${i})">Registrar Corte</button>`;
    html += `<button onclick="renovarPlano(${i})">Renovar Plano</button>`;
    html += `<button onclick="togglePago(${i})">Marcar/Desmarcar Pago</button>`;
    html += `<button onclick="editarCliente(${i})">Editar</button>`;
    html += `<button onclick="excluirCliente(${i})">Excluir</button>`;
    html += `<button onclick="enviarCobranca(${i})">Enviar Cobrança</button>`;
    html += `</td></tr>`;
  });
  html += '</tbody></table>';
  wrap.innerHTML = html;

  // totais
  let totalRecebido=0, totalPendente=0, cortesFeitos=0, cortesRest=0;
  for(const c of clients){ cortesFeitos += c.cortesFeitos; cortesRest += Math.max(0,c.cortesMes - c.cortesFeitos); if(c.pago) totalRecebido += c.valor; else totalPendente += c.valor; }
  document.getElementById('totais').innerText = `Recebido: ${formatCurrency(totalRecebido)} • Pendente: ${formatCurrency(totalPendente)} • Cortes feitos: ${cortesFeitos} • Restantes: ${cortesRest}`;
}

// Init
function init(){ renderTable(); atualizarChart(); carregarMeses(); }
init();
</script>
</body>
</html>

