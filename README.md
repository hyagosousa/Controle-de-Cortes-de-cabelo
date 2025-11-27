
<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Barber Control — Hyago</title>
<style>
  /* Tema preto + azul, botão confirmar em verde */
  :root{
    --bg:#050505;
    --card:#0f1220;
    --blue:#1e90ff;
    --accent:#4da3ff;
    --muted:#9aa4b2;
    --confirm:#0f9d58;
  }
  html,body{height:100%;margin:0;background:var(--bg);color:#e6eef6;font-family:Inter,Arial,Helvetica,sans-serif}
  .wrap{max-width:960px;margin:12px auto;padding:12px}
  header{display:flex;align-items:center;justify-content:space-between;gap:12px;margin-bottom:12px}
  header h1{margin:0;color:var(--accent)}
  .card{background:var(--card);padding:14px;border-radius:12px;margin-bottom:12px;border:1px solid rgba(30,144,255,0.06);box-shadow:0 6px 18px rgba(0,0,0,0.6)}
  label{display:block;font-size:13px;color:var(--muted);margin-bottom:6px}
  input,select,button{width:100%;padding:10px;border-radius:10px;border:1px solid #222;background:#0b0b12;color:#e6eef6;font-size:15px}
  .row{display:flex;gap:8px;flex-wrap:wrap}
  .row > *{flex:1}
  .small{font-size:13px;color:var(--muted);margin-top:6px}
  button.primary{background:linear-gradient(90deg,var(--blue),#0066cc);color:white;border:none;cursor:pointer}
  button.confirm{background:var(--confirm);color:#fff;border:none;cursor:pointer}
  .clienteBox{background:#0b0c12;padding:12px;border-radius:10px;border-left:4px solid var(--blue);margin-top:10px;box-shadow:0 4px 10px rgba(0,0,0,0.5)}
  .clienteHead{display:flex;justify-content:space-between;align-items:center;gap:8px}
  .clienteName{font-weight:700}
  .progressBar{height:12px;background:#121217;border-radius:999px;margin-top:8px;overflow:hidden;border:1px solid #111}
  .progress{height:100%;background:linear-gradient(90deg,#00c8ff,#1e90ff);width:0%;transition:width .25s}
  .actions{display:flex;gap:8px;margin-top:10px;flex-wrap:wrap}
  .btn-small{padding:8px;border-radius:8px;border:1px solid rgba(255,255,255,0.04);background:#071026;color:#e6eef6;cursor:pointer}
  .btn-danger{background:#8b1d1d}
  footer{margin-top:18px;text-align:center;color:var(--muted);font-size:13px}
  @media(min-width:900px){ .row > input[type="text"]{flex:2} .row > input[type="number"]{flex:1} }
</style>
</head>
<body>
<div class="wrap">
  <header>
    <h1>Barber Control — Hyago</h1>
    <div class="small">Tema: preto + azul • Botão pagar: verde</div>
  </header>

  <div class="card">
    <h3>➕ Adicionar Cliente</h3>
    <div class="row" style="margin-top:8px">
      <input id="nome" type="text" placeholder="Nome do cliente" />
      <input id="telefone" type="text" placeholder="Telefone (somente números, ex: 5511999998888)" />
    </div>
    <div class="row" style="margin-top:8px">
      <input id="valorPlano" type="number" step="0.01" placeholder="Valor do plano (R$)" />
      <input id="totalCortes" type="number" placeholder="Cortes por plano (ex: 2)" />
    </div>
    <div style="display:flex;gap:8px;margin-top:10px">
      <button class="primary" onclick="addCliente()">Adicionar</button>
      <button class="btn-small" onclick="exportarCSV()">Exportar CSV</button>
    </div>
    <div class="small">Preencha nome, valor e total. Telefone é opcional (usado para WhatsApp ao finalizar).</div>
  </div>

  <div class="card">
    <h3>🔎 Buscar / Filtrar</h3>
    <input id="busca" type="text" placeholder="Buscar por nome..." oninput="renderClientes()" />
    <div class="small">Digite parte do nome para filtrar a lista.</div>
  </div>

  <div id="listaClientes"></div>

  <div class="card" style="display:flex;gap:8px;flex-wrap:wrap;align-items:center">
    <button class="primary" onclick="exportarPDF()">📄 Exportar histórico (TXT)</button>
    <button class="primary" onclick="enviarRelatorioWhatsApp()">📩 Enviar relatório (WhatsApp)</button>
    <button class="btn-danger" onclick="limparTudo()">🗑 Limpar tudo</button>
  </div>

  <div class="card" id="dashboardCard">
    <h3>📊 Dashboard</h3>
    <div id="dashResumo" class="small"></div>
  </div>

  <footer>Dados salvos no navegador (localStorage). Faça backups exportando CSV/arquivo.</footer>
</div>

<script>
/* Storage key */
const KEY = 'barber_cortes_v4';

/* State */
let clientes = JSON.parse(localStorage.getItem(KEY) || '[]');

/* Helpers */
function salvar(){ localStorage.setItem(KEY, JSON.stringify(clientes)); }
function formatCurrency(v){ return 'R$ ' + Number(v||0).toFixed(2).replace('.',','); }

/* Add client */
function addCliente(){
  const nome = document.getElementById('nome').value.trim();
  const telefone = document.getElementById('telefone').value.trim();
  const valorPlano = Number(document.getElementById('valorPlano').value);
  const totalCortes = Number(document.getElementById('totalCortes').value);

  if(!nome || !isFinite(valorPlano) || valorPlano < 0 || !Number.isFinite(totalCortes) || totalCortes <= 0){
    return alert('Preencha corretamente: nome, valor do plano e total de cortes.');
  }

  clientes.push({
    nome,
    telefone,
    valorPlano: Number(valorPlano.toFixed(2)),
    totalCortes: Math.floor(totalCortes),
    feitos: 0,
    pago: false,
    historico: []
  });

  salvar();
  document.getElementById('nome').value = '';
  document.getElementById('telefone').value = '';
  document.getElementById('valorPlano').value = '';
  document.getElementById('totalCortes').value = '';
  renderClientes();
}

/* Registrar corte */
function registrarCorte(i){
  const c = clientes[i];
  if(!c) return;
  if(c.feitos >= c.totalCortes){ alert('Plano já finalizado.'); return; }

  c.feitos++;
  c.historico.unshift({ when: new Date().toLocaleString(), type: 'corte' });

  // somente quando finalizar o plano: enviar mensagem pelo WhatsApp (se tiver telefone)
  if(c.feitos === c.totalCortes){
    if(c.telefone){
      const msg = `Olá ${c.nome}! Você finalizou todos os cortes do seu plano. Obrigado!`;
      window.open(`https://wa.me/${c.telefone}?text=${encodeURIComponent(msg)}`, '_blank');
    } else {
      // sem telefone, só notifica no app
      alert(`Cliente ${c.nome} finalizou o plano.`);
    }
  }

  salvar();
  renderClientes();
}

/* Confirmar pagamento (botão verde) */
function confirmarPagamento(i){
  const c = clientes[i];
  if(!c) return;
  c.pago = true;
  c.historico.unshift({ when: new Date().toLocaleString(), type: 'pagamento' });
  salvar();
  renderClientes();
}

/* Renovar plano (resetar feitos e marcar não pago) */
function renovarPlano(i){
  if(!confirm('Renovar plano para ' + clientes[i].nome + '?')) return;
  clientes[i].feitos = 0;
  clientes[i].pago = false;
  clientes[i].historico.unshift({ when: new Date().toLocaleString(), type: 'renovado' });
  salvar();
  renderClientes();
}

/* Excluir cliente */
function excluirCliente(i){
  if(!confirm('Excluir cliente '+clientes[i].nome+' ?')) return;
  clientes.splice(i,1);
  salvar();
  renderClientes();
}

/* Mostrar histórico */
function mostrarHistorico(i){
  const c = clientes[i];
  if(!c) return;
  if(!c.historico || c.historico.length===0) return alert('Nenhum registro.');
  let texto = `Histórico — ${c.nome}\n\n`;
  c.historico.forEach((h,idx)=> texto += `${idx+1}. ${h.when} — ${h.type}\n`);
  alert(texto);
}

/* Render lista */
function renderClientes(){
  const q = (document.getElementById('busca').value || '').toLowerCase();
  const wrap = document.getElementById('listaClientes');
  wrap.innerHTML = '';

  if(clientes.length === 0){
    wrap.innerHTML = `<div class="card small">Nenhum cliente cadastrado.</div>`;
    atualizarDashboard();
    return;
  }

  clientes.forEach((c,i)=>{
    if(q && !c.nome.toLowerCase().includes(q)) return;

    const pct = Math.round((c.feitos / c.totalCortes) * 100) || 0;

    const tpl = document.createElement('div');
    tpl.className = 'clienteBox';
    tpl.innerHTML = `
      <div class="clienteHead">
        <div>
          <div class="clienteName">${c.nome}</div>
          <div class="small">${c.telefone ? c.telefone : ''}</div>
        </div>
        <div style="text-align:right">
          <div class="small">${c.feitos}/${c.totalCortes} cortes</div>
          <div class="small">${formatCurrency(c.valorPlano)}</div>
        </div>
      </div>
      <div class="progressBar"><div class="progress" style="width:${pct}%"></div></div>
      <div class="actions">
        <button class="btn-small" onclick="registrarCorte(${i})">Registrar Corte</button>
        <button class="btn-small" onclick="mostrarHistorico(${i})">Histórico</button>
        <button class="btn-small" onclick="renovarPlano(${i})">Renovar</button>
        <button class="btn-small confirm" onclick="confirmarPagamento(${i})">Confirmar Pagamento</button>
        <button class="btn-small" onclick="enviarCobranca(${i})">Cobrar (WhatsApp)</button>
        <button class="btn-small" style="background:#6b0000" onclick="excluirCliente(${i})">Excluir</button>
      </div>
    `;
    wrap.appendChild(tpl);
  });

  atualizarDashboard();
}

/* Cobrança via WhatsApp — usa valor do plano */
function enviarCobranca(i){
  const c = clientes[i];
  if(!c) return alert('Cliente inválido');
  if(!c.telefone) return alert('Cliente sem telefone cadastrado.');
  const chavePIX = 'hyagosousasous@gmail.com';
  const texto = `Olá ${c.nome}! Segue o lembrete de pagamento: ${formatCurrency(c.valorPlano)}. PIX: ${chavePIX}`;
  window.open(`https://wa.me/${c.telefone}?text=${encodeURIComponent(texto)}`, '_blank');
}

/* Export CSV */
function exportarCSV(){
  if(clientes.length === 0) return alert('Sem dados para exportar');
  const rows = [['Nome','Telefone','ValorPlano','TotalCortes','Feitos','Pago']];
  clientes.forEach(c => rows.push([c.nome,c.telefone,c.valorPlano,c.totalCortes,c.feitos,c.pago? 'SIM':'NÃO']));
  const csv = rows.map(r => r.map(v => `"${String(v).replace(/"/g,'""')}"`).join(',')).join('\n');
  const blob = new Blob([csv],{type:'text/csv;charset=utf-8;'});
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a'); a.href = url; a.download = 'barber_clients.csv'; a.click(); URL.revokeObjectURL(url);
}

/* Export simple TXT (historico) */
function exportarPDF(){
  const txt = JSON.stringify(clientes,null,2);
  const blob = new Blob([txt], { type: 'text/plain' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a'); a.href = url; a.download = 'historico_cortes.txt'; a.click(); URL.revokeObjectURL(url);
}

/* Enviar relatório resumido por WhatsApp (abrir WhatsApp Web) */
function enviarRelatorioWhatsApp(){
  if(clientes.length === 0) return alert('Sem clientes');
  let txt = `Relatório de cortes — ${new Date().toLocaleDateString()}%0A%0A`;
  clientes.forEach(c => txt += `${c.nome}: ${c.feitos}/${c.totalCortes} — ${formatCurrency(c.valorPlano)}%0A`);
  window.open(`https://web.whatsapp.com/send?text=${encodeURIComponent(txt)}`, '_blank');
}

/* Dashboard */
function atualizarDashboard(){
  const totalClientes = clientes.length;
  const totalCortes = clientes.reduce((s,c) => s + (c.totalCortes||0), 0);
  const totalFeitos = clientes.reduce((s,c) => s + (c.feitos||0), 0);
  const totalReceita = clientes.reduce((s,c) => s + (c.pago ? (c.valorPlano||0) : 0), 0);
  const progress = totalCortes ? Math.round((totalFeitos / totalCortes) * 100) : 0;
  document.getElementById('dashResumo').innerHTML = `
    <div class="small">Clientes: <b>${totalClientes}</b></div>
    <div class="small">Cortes contratados: <b>${totalCortes}</b></div>
    <div class="small">Cortes realizados: <b>${totalFeitos}</b></div>
    <div class="small">Progresso geral: <b>${progress}%</b></div>
    <div class="small">Receita confirmada: <b>${formatCurrency(totalReceita)}</b></div>
  `;
}

/* Clear all */
function limparTudo(){
  if(!confirm('Apagar TODOS os dados?')) return;
  clientes = [];
  salvar();
  renderClientes();
}

/* Init */
renderClientes();
</script>
</body>
</html>
