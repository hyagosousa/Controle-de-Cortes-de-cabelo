<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Barber Control — Hyago (Mobile)</title>
<!-- Chart.js & jsPDF & html2canvas -->
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/html2canvas/1.4.1/html2canvas.min.js"></script>
<style>
  :root{--bg:#f3f5f7;--card:#ffffff;--accent:#1f8f49;--muted:#6b7280}
  *{box-sizing:border-box}
  body{margin:0;font-family:Inter, system-ui, -apple-system, 'Segoe UI', Roboto, Arial;background:var(--bg);color:#111}
  .wrap{max-width:900px;margin:0 auto;padding:12px}
  header{display:flex;align-items:center;justify-content:space-between;margin-bottom:10px}
  h1{font-size:18px;margin:0;color:var(--accent)}
  .card{background:var(--card);padding:12px;border-radius:12px;box-shadow:0 6px 18px rgba(2,6,23,0.06);margin-bottom:12px}
  .row{display:flex;gap:8px}
  input,select,button{font-size:16px}
  input[type="text"], input[type="number"]{flex:1;padding:10px;border-radius:8px;border:1px solid #ddd}
  button{padding:10px 12px;border-radius:8px;border:0;background:var(--accent);color:#fff;cursor:pointer}
  button.ghost{background:transparent;border:1px solid #ddd;color:var(--accent)}
  .cliente{padding:10px;border-radius:10px;border:1px solid #eee;margin-bottom:8px;display:flex;flex-direction:column}
  .clienteHead{display:flex;align-items:center;justify-content:space-between}
  .clienteName{font-weight:700}
  .small{font-size:13px;color:var(--muted)}
  .actions{display:flex;gap:6px;margin-top:8px;flex-wrap:wrap}
  .btnRed{background:#e11d48}
  .progressWrap{height:10px;background:#eee;border-radius:999px;overflow:hidden;margin-top:8px}
  .progressBar{height:100%;background:var(--accent);width:0%}
  @media(min-width:800px){ .row{flex-wrap:nowrap} }
</style>
</head>
<body>
<div class="wrap">
  <header>
    <h1>Barber Control — Hyago</h1>
    <div class="small">Mobile friendly</div>
  </header>

  <div class="card">
    <div style="display:flex;gap:8px;margin-bottom:8px;align-items:center">
      <input id="search" type="text" placeholder="Buscar cliente por nome..." oninput="listarClientes()">
      <button class="ghost" onclick="mostrarResumo()">Resumo</button>
    </div>

    <form id="form" onsubmit="event.preventDefault();addCliente()">
      <div style="display:flex;gap:8px;flex-wrap:wrap">
        <input id="nome" type="text" placeholder="Nome do cliente">
        <input id="telefone" type="text" placeholder="WhatsApp (somente números)" style="width:180px">
        <input id="cortes" type="number" placeholder="Cortes/mês" style="width:120px" value="2">
        <input id="valor" type="number" step="0.01" placeholder="Valor R$" style="width:120px" value="80.00">
      </div>
      <div style="display:flex;gap:8px;margin-top:8px">
        <button type="submit">Adicionar</button>
        <button type="button" class="ghost" onclick="exportarCSV()">Exportar CSV</button>
        <button type="button" class="ghost" onclick="exportarPDFResumo()">Exportar PDF</button>
      </div>
      <div class="small" style="margin-top:8px">Preencha telefone com DDI (ex: 5511999998888). Valor em reais.</div>
    </form>
  </div>

  <div id="lista" class="card"></div>

  <div class="card">
    <h3 style="margin:0 0 8px 0">Dashboard</h3>
    <canvas id="chart" height="120"></canvas>
    <div style="display:flex;gap:8px;margin-top:10px;flex-wrap:wrap">
      <button class="ghost" onclick="fecharMes()">Fechar mês (salvar histórico)</button>
      <button class="ghost" onclick="enviarRelatorioWhatsApp()">Enviar relatório (WhatsApp)</button>
      <button class="ghost btnRed" onclick="limparTudo()">Limpar todos os dados</button>
    </div>
  </div>

  <div id="histArea" class="card" style="display:none"></div>
</div>

<script>
// storage keys
const KEY = 'barber_v3_clients';
const KEY_HISTORY = 'barber_v3_history';

let clients = JSON.parse(localStorage.getItem(KEY)) || [];
let history = JSON.parse(localStorage.getItem(KEY_HISTORY)) || {}; // { '2025-11': { date:'2025-11', snapshot:[], totals:{} } }

// helpers
function salvar(){ localStorage.setItem(KEY, JSON.stringify(clients)); }
function salvarHist(){ localStorage.setItem(KEY_HISTORY, JSON.stringify(history)); }
function formatCurrency(v){ return 'R$ ' + Number(v).toFixed(2).replace('.',','); }

// UI: add client
function addCliente(){
  const nome = document.getElementById('nome').value.trim();
  const telefone = document.getElementById('telefone').value.trim();
  const cortes = Number(document.getElementById('cortes').value) || 2;
  const valor = Number(document.getElementById('valor').value) || 0;
  if(!nome){ alert('Digite o nome'); return; }
  clients.push({ nome, telefone, cortesMes: cortes, cortesFeitos:0, valor, historico:[] });
  salvar(); document.getElementById('form').reset(); listarClientes(); atualizarChart();
}

// render list
function listarClientes(){
  const q = (document.getElementById('search').value||'').toLowerCase();
  const wrap = document.getElementById('lista'); wrap.innerHTML='';
  if(clients.length===0){ wrap.innerHTML = '<div class="small">Nenhum cliente cadastrado.</div>'; return; }
  clients.forEach((c,i)=>{
    if(q && !c.nome.toLowerCase().includes(q)) return;
    const rest = Math.max(0,c.cortesMes - c.cortesFeitos);
    const percent = Math.round((c.cortesFeitos / c.cortesMes) * 100);
    const div = document.createElement('div'); div.className='cliente';
    div.innerHTML = `
      <div class="clienteHead">
        <div>
          <div class="clienteName">${c.nome}</div>
          <div class="small">${c.telefone||''}</div>
        </div>
        <div style="text-align:right">
          <div class="small">${c.cortesFeitos}/${c.cortesMes} cortes</div>
          <div class="small">${formatCurrency(c.valor)}</div>
        </div>
      </div>
      <div class="progressWrap"><div class="progressBar" style="width:${percent}%"></div></div>
      <div class="actions">
        <button onclick="registrarCorte(${i})">Registrar 1 corte</button>
        <button class="ghost" onclick="mostrarHistorico(${i})">Histórico</button>
        <button class="ghost" onclick="enviarCobranca(${i})">Cobrar (WhatsApp)</button>
        <button class="ghost" onclick="renovarPlano(${i})">Renovar plano</button>
        <button class="btnRed" onclick="excluirCliente(${i})">Excluir</button>
      </div>
    `;
    wrap.appendChild(div);
  });
}

// registrar corte + avisos + historico
function registrarCorte(i){
  const c = clients[i];
  if(!c) return;
  if(c.cortesFeitos >= c.cortesMes){ alert('Plano já completo. Renove o plano.'); return; }
  c.cortesFeitos += 1;
  const now = new Date().toLocaleString();
  c.historico.unshift({ when: now, action: 'corte' });
  salvar(); listarClientes(); atualizarChart();
  const faltam = c.cortesMes - c.cortesFeitos;
  if(faltam === 1){ // aviso e abrir WhatsApp com mensagem pronta
    if(c.telefone){
      const msg = `Olá ${c.nome}!%0A%0AFalta apenas 1 corte do seu pacote. Quer agendar?`;
      const url = `https://wa.me/${c.telefone}?text=${msg}`;
      window.open(url,'_blank');
    } else {
      alert('Falta 1 corte — mas cliente não possui telefone cadastrado.');
    }
  }
  if(faltam === 0){ alert('Plano finalizado — notifique para renovar se quiser.'); }
}

function renovarPlano(i){ if(!confirm('Renovar plano para '+clients[i].nome+' ?')) return; clients[i].cortesFeitos = 0; clients[i].historico.unshift({ when:new Date().toLocaleString(), action:'renovado' }); salvar(); listarClientes(); atualizarChart(); }

function excluirCliente(i){ if(!confirm('Excluir '+clients[i].nome+' ?')) return; clients.splice(i,1); salvar(); listarClientes(); atualizarChart(); }

function mostrarHistorico(i){
  const c = clients[i];
  const area = document.getElementById('histArea'); area.style.display='block';
  let html = `<h3>Histórico — ${c.nome}</h3>`;
  if(!c.historico || c.historico.length===0) html += '<div class="small">Nenhum registro.</div>';
  else html += '<ul>' + c.historico.map(h => `<li>${h.when} — ${h.action}</li>`).join('') + '</ul>';
  html += '<div style="margin-top:8px"><button class="ghost" onclick="hideHistorico()">Fechar histórico</button></div>';
  area.innerHTML = html; area.scrollIntoView({behavior:'smooth'});
}
function hideHistorico(){ document.getElementById('histArea').style.display='none'; }

// cobrança via whatsapp
function enviarCobranca(i){ const c = clients[i]; if(!c.telefone) return alert('Cliente sem telefone'); const chave='hyagosousasous@gmail.com'; const texto = `Olá ${c.nome}!%0A%0APor favor, pagamento do plano: ${formatCurrency(c.valor)}.%0APIX: ${chave}`; window.open(`https://wa.me/${c.telefone}?text=${texto}`,'_blank'); }

// Resumo geral (tela rápida)
function mostrarResumo(){ const totalClientes = clients.length; let totalRecebido=0,totalPendente=0,totalCortes=0,feitos=0; clients.forEach(c=>{ totalCortes += c.cortesMes; feitos += c.cortesFeitos; if(c.pago) totalRecebido += c.valor; else totalPendente += c.valor; }); const faltam = totalCortes - feitos; alert(`Clientes: ${totalClientes}
Cortes feitos: ${feitos}
Cortes restantes: ${faltam}
Recebido: ${formatCurrency(totalRecebido)}
Pendente: ${formatCurrency(totalPendente)}`); }

// fechar mês -> salvar snapshot
function fecharMes(){ if(!confirm('Salvar histórico do mês e resetar cortes?')) return; const key = new Date().toISOString().slice(0,7); const snapshot = clients.map(c=>({ nome:c.nome, cortesFeitos:c.cortesFeitos, cortesMes:c.cortesMes, valor:c.valor, historico:c.historico })); history[key] = { date:key, snapshot, totals: calcTotals(clients) }; salvarHist(); clients = clients.map(c=>({...c,cortesFeitos:0})); salvar(); listarClientes(); atualizarChart(); carregarMeses(); alert('Mês salvo: '+key); }

function calcTotals(list){ let totalRecebido=0,totalPendente=0,cortesFeitos=0,cortesRest=0; for(const c of list){ cortesFeitos += c.cortesFeitos; cortesRest += (c.cortesMes - c.cortesFeitos); if(c.pago) totalRecebido += c.valor; else totalPendente += c.valor; } return { totalRecebido,totalPendente,cortesFeitos,cortesRest }; }

// export CSV (clients)
function exportarCSV(){ if(clients.length===0) return alert('Sem dados'); const rows=[['Nome','Telefone','CortesFeitos','CortesMes','Valor','Pago']]; clients.forEach(c=>rows.push([c.nome,c.telefone,c.cortesFeitos,c.cortesMes,c.valor,!!c.pago])); const csv = rows.map(r=>r.map(v=>`"${String(v).replace(/"/g,'""')}"`).join(',')).join('
'); const blob=new Blob([csv],{type:'text/csv'}); const url=URL.createObjectURL(blob); const a=document.createElement('a'); a.href=url; a.download='barber_clients.csv'; a.click(); URL.revokeObjectURL(url); }

// export PDF resumo (uses html2canvas + jsPDF)
async function exportarPDFResumo(){
  // build a simple DOM fragment for the report
  const div = document.createElement('div'); div.style.padding='10px'; div.style.fontSize='12px'; div.innerHTML = `<h2>Relatório — ${new Date().toLocaleDateString()}</h2>`;
  clients.forEach(c => { div.innerHTML += `<div style="margin-bottom:6px"><b>${c.nome}</b> — ${c.cortesFeitos}/${c.cortesMes} cortes — ${formatCurrency(c.valor)}</div>`; });
  document.body.appendChild(div);
  const canvas = await html2canvas(div, { scale:2 });
  const imgData = canvas.toDataURL('image/png');
  const { jsPDF } = window.jspdf;
  const pdf = new jsPDF({ orientation:'portrait', unit:'pt', format:'a4' });
  const imgProps = pdf.getImageProperties(imgData);
  const pdfWidth = pdf.internal.pageSize.getWidth();
  const pdfHeight = (imgProps.height * pdfWidth) / imgProps.width;
  pdf.addImage(imgData,'PNG',0,20,pdfWidth,pdfHeight);
  pdf.save('relatorio_barber_'+new Date().toISOString().slice(0,10)+'.pdf');
  document.body.removeChild(div);
}

// enviar relatório completo via WhatsApp (texto resumido)
function enviarRelatorioWhatsApp(){ if(clients.length===0) return alert('Sem clientes'); let text = `Relatório rápido — ${new Date().toLocaleDateString()}%0A%0A`; clients.forEach(c=>{ text += `${c.nome}: ${c.cortesFeitos}/${c.cortesMes} cortes — ${formatCurrency(c.valor)}%0A`; }); // open wa.me without phone -> user can pick contact via WhatsApp Web
  const url = `https://web.whatsapp.com/send?text=${encodeURIComponent(text)}`; window.open(url,'_blank'); }

// history UI helpers
function carregarMeses(){ const sel = document.getElementById('selMes'); if(!sel) return; sel.innerHTML=''; const keys = Object.keys(history).sort().reverse(); const opt = document.createElement('option'); opt.value=''; opt.textContent='Selecionar mês'; sel.appendChild(opt); keys.forEach(k=>{ const o=document.createElement('option'); o.value=k; o.textContent=k; sel.appendChild(o); }); }

function downloadHistoricoCSV(){ const sel=document.getElementById('selMes'); if(!sel) return alert('Nada a baixar'); const key=sel.value; if(!key) return alert('Selecione mês'); const h = history[key]; if(!h) return alert('Mês não encontrado'); const rows=[['Nome','CortesFeitos','CortesMes','Valor']]; h.snapshot.forEach(r=> rows.push([r.nome,r.cortesFeitos,r.cortesMes,r.valor])); const csv = rows.map(r=>r.map(v=>`"${String(v).replace(/"/g,'""')}"`).join(',')).join('
'); const blob=new Blob([csv],{type:'text/csv'}); const url=URL.createObjectURL(blob); const a=document.createElement('a'); a.href=url; a.download=`barber_history_${key}.csv`; a.click(); URL.revokeObjectURL(url); }

function limparTudo(){ if(!confirm('Apagar todos os dados (clientes e histórico)?')) return; clients=[]; history={}; salvar(); salvarHist(); listarClientes(); atualizarChart(); carregarMeses(); alert('Dados apagados'); }

// Chart
let chart=null;
function atualizarChart(){ const labels = clients.map(c=>c.nome); const feitos = clients.map(c=>c.cortesFeitos); const restantes = clients.map(c=>Math.max(0,c.cortesMes - c.cortesFeitos)); const ctx = document.getElementById('chart').getContext('2d'); const data = { labels, datasets:[ { label:'Feitos', data:feitos }, { label:'Restantes', data:restantes } ] }; if(chart){ chart.data = data; chart.update(); } else { chart = new Chart(ctx,{ type:'bar', data, options:{ responsive:true, scales:{ y:{ beginAtZero:true, precision:0 } } } }); } }

// init
listarClientes(); atualizarChart(); carregarMeses();
</script>
</body>
</html>

