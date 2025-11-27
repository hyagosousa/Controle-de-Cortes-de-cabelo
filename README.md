<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Barber Simple — Controle de Cortes</title>
<style>
  :root{--bg:#050505;--card:#0f1220;--blue:#1e90ff;--muted:#9aa4b2;--green:#0f9d58}
  html,body{height:100%;margin:0;background:var(--bg);color:#e6eef6;font-family:Arial,Helvetica,sans-serif}
  .wrap{max-width:880px;margin:12px auto;padding:12px}
  h1{margin:0 0 10px;color:var(--blue);text-align:center}
  .card{background:var(--card);padding:12px;border-radius:10px;margin-bottom:12px;border:1px solid rgba(30,144,255,0.06)}
  input,button,select{width:100%;padding:10px;border-radius:8px;border:1px solid #222;background:#0b0b12;color:#e6eef6;font-size:15px;margin-top:8px}
  .row{display:flex;gap:8px;flex-wrap:wrap}
  .row > *{flex:1}
  .small{font-size:13px;color:var(--muted);margin-top:6px}
  .cliente{background:#0b0c12;padding:10px;border-radius:8px;margin-top:10px;border-left:4px solid var(--blue)}
  .cliente h4{margin:0 0 6px}
  .progress{height:10px;background:#111;border-radius:8px;overflow:hidden;margin:8px 0}
  .bar{height:100%;background:linear-gradient(90deg,#00c8ff,#1e90ff);width:0%}
  .actions{display:flex;gap:8px;flex-wrap:wrap;margin-top:6px}
  .btn{padding:8px;border-radius:8px;border:none;cursor:pointer;background:#004f99;color:white}
  .btn-green{background:var(--green)}
  .btn-danger{background:#8b1d1d}
  footer{font-size:13px;text-align:center;color:var(--muted);margin-top:12px}
  @media(min-width:800px){ .row > input[type="text"]{flex:2} .row > input[type="number"]{flex:1} }
</style>
</head>
<body>
<div class="wrap">
  <h1>Barber Simple — Controle de Cortes</h1>

  <div class="card">
    <strong>➕ Adicionar cliente</strong>
    <div class="row">
      <input id="nome" type="text" placeholder="Nome completo" />
      <input id="telefone" type="text" placeholder="Telefone (ex: 5511999998888) - opcional" />
    </div>
    <div class="row">
      <input id="valor" type="number" step="0.01" placeholder="Valor do plano (R$)" />
      <input id="cortes" type="number" placeholder="Cortes no plano (ex: 2)" />
    </div>
    <div style="display:flex;gap:8px;margin-top:8px">
      <button class="btn" onclick="addCliente()">Adicionar</button>
      <button class="btn" onclick="exportCSV()">Exportar CSV</button>
    </div>
    <div class="small">Telefone opcional — usado para enviar WhatsApp quando finalizar ou cobrar.</div>
  </div>

  <div class="card">
    <strong>🔎 Buscar</strong>
    <input id="buscar" type="text" placeholder="Buscar por nome..." oninput="render()" />
  </div>

  <div id="lista"></div>

  <div class="card">
    <strong>📊 Resumo</strong>
    <div id="resumo" class="small" style="margin-top:8px"></div>
  </div>

  <footer>Dados salvos localmente no navegador. Faça backup exportando CSV.</footer>
</div>

<script>
/* ---------- state ---------- */
const KEY = 'barber_simple_db_v1';
let clientes = JSON.parse(localStorage.getItem(KEY) || '[]');

/* ---------- helpers ---------- */
function save(){ localStorage.setItem(KEY, JSON.stringify(clientes)); }
function money(v){ return 'R$ ' + Number(v||0).toFixed(2).replace('.',','); }
function sanitizePhone(raw){ return String(raw||'').replace(/\D/g,'').replace(/^0+/,''); }

/* ---------- add cliente ---------- */
function addCliente(){
  const nome = document.getElementById('nome').value.trim();
  const telefone = sanitizePhone(document.getElementById('telefone').value);
  const valor = Number(document.getElementById('valor').value);
  const cortes = Number(document.getElementById('cortes').value);

  if(!nome || !isFinite(valor) || valor < 0 || !isFinite(cortes) || cortes <= 0){
    return alert('Preencha corretamente: nome, valor e cortes.');
  }

  clientes.push({
    nome,
    telefone,            // pode ser vazio
    valorPlano: +valor.toFixed(2),
    totalCortes: Math.floor(cortes),
    feitos: 0,
    pago: false,
    historico: []        // array de timestamps
  });

  save();
  document.getElementById('nome').value = '';
  document.getElementById('telefone').value = '';
  document.getElementById('valor').value = '';
  document.getElementById('cortes').value = '';
  render();
}

/* ---------- registrar corte ---------- */
function registrarCorte(i){
  const c = clientes[i];
  if(!c) return;
  if(c.feitos >= c.totalCortes){ return alert('Plano já finalizado.'); }

  c.feitos++;
  c.historico.unshift(new Date().toLocaleString());
  // se finalizar, abrir WhatsApp (se tiver telefone)
  if(c.feitos === c.totalCortes){
    if(c.telefone){
      const msg = `Olá ${c.nome}! Você finalizou seu plano de cortes. Deseja renovar?`;
      window.open(`https://wa.me/${c.telefone}?text=${encodeURIComponent(msg)}`, '_blank');
    } else {
      alert(`Cliente ${c.nome} finalizou o plano.`);
    }
  }
  save();
  render();
}

/* ---------- confirmar pagamento ---------- */
function confirmarPagamento(i){
  const c = clientes[i];
  if(!c) return;
  c.pago = true;
  c.historico.unshift(new Date().toLocaleString() + ' — pagamento');
  save();
  render();
}

/* ---------- cobrar (abre WhatsApp com mensagem + PIX) ---------- */
function cobrar(i){
  const c = clientes[i];
  if(!c) return alert('Cliente inválido.');
  const chavePIX = 'hyagosousasous@gmail.com'; // sua chave PIX
  const telefone = c.telefone;
  const msg = `Olá ${c.nome}, segue lembrete de pagamento: ${money(c.valorPlano)}. PIX: ${chavePIX}`;
  // se telefone disponível, abrir direto; se não, abrir WhatsApp Web sem telefone (usuário escolhe)
  if(telefone){
    window.open(`https://wa.me/${telefone}?text=${encodeURIComponent(msg)}`, '_blank');
  } else {
    window.open(`https://web.whatsapp.com/send?text=${encodeURIComponent(msg)}`, '_blank');
  }
}

/* ---------- histórico ---------- */
function mostrarHistorico(i){
  const c = clientes[i];
  if(!c) return;
  if(!c.historico.length) return alert('Nenhum registro ainda.');
  alert('Histórico — ' + c.nome + '\n\n' + c.historico.join('\n'));
}

/* ---------- renovar plano ---------- */
function renovar(i){
  if(!confirm('Renovar plano para ' + clientes[i].nome + '?')) return;
  clientes[i].feitos = 0;
  clientes[i].pago = false;
  clientes[i].historico.unshift(new Date().toLocaleString() + ' — renovado');
  save();
  render();
}

/* ---------- excluir ---------- */
function excluir(i){
  if(!confirm('Excluir cliente ' + clientes[i].nome + '?')) return;
  clientes.splice(i,1);
  save();
  render();
}

/* ---------- export CSV ---------- */
function exportCSV(){
  if(!clientes.length) return alert('Sem dados para exportar.');
  const rows = [['Nome','Telefone','ValorPlano','TotalCortes','Feitos','Pago','Histórico']];
  clientes.forEach(c=>{
    rows.push([c.nome,c.telefone||'',c.valorPlano,c.totalCortes,c.feitos,c.pago? 'SIM':'NÃO', (c.historico||[]).join(' | ')]);
  });
  const csv = rows.map(r=>r.map(v=>`"${String(v).replace(/"/g,'""')}"`).join(',')).join('\n');
  const blob = new Blob([csv],{type:'text/csv;charset=utf-8;'});
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url; a.download = 'barber_clients.csv'; a.click(); URL.revokeObjectURL(url);
}

/* ---------- render UI ---------- */
function render(){
  const q = (document.getElementById('buscar')?.value || '').toLowerCase();
  const wrap = document.getElementById('lista');
  wrap.innerHTML = '';

  if(!clientes.length){
    wrap.innerHTML = '<div class="card small">Nenhum cliente cadastrado.</div>';
    atualizarResumo();
    return;
  }

  clientes.forEach((c,i)=>{
    if(q && !c.nome.toLowerCase().includes(q)) return;

    const pct = Math.round((c.feitos / c.totalCortes) * 100) || 0;
    const div = document.createElement('div');
    div.className = 'cliente';
    div.innerHTML = `
      <h4>${c.nome}</h4>
      <div class="small">${c.telefone ? c.telefone : ''}</div>
      <div class="small">Plano: ${money(c.valorPlano)} • ${c.feitos}/${c.totalCortes} cortes</div>
      <div class="progress"><div class="bar" style="width:${pct}%"></div></div>
      <div class="actions">
        <button class="btn" onclick="registrarCorte(${i})">Registrar corte</button>
        <button class="btn btn-green" onclick="confirmarPagamento(${i})">Confirmar pagamento</button>
        <button class="btn" onclick="cobrar(${i})">Cobrar (WhatsApp)</button>
        <button class="btn" onclick="mostrarHistorico(${i})">Histórico</button>
        <button class="btn" onclick="renovar(${i})">Renovar</button>
        <button class="btn btn-danger" onclick="excluir(${i})">Excluir</button>
      </div>
    `;
    wrap.appendChild(div);
  });

  atualizarResumo();
}

/* ---------- resumo/dashboard ---------- */
function atualizarResumo(){
  const totalClientes = clientes.length;
  const totalContratados = clientes.reduce((s,c)=> s + (c.totalCortes||0), 0);
  const totalFeitos = clientes.reduce((s,c)=> s + (c.feitos||0), 0);
  const receitaConfirmada = clientes.reduce((s,c)=> s + ((c.pago? c.valorPlano:0)||0), 0);
  document.getElementById('resumo').innerHTML = `
    <div class="small">Clientes: <b>${totalClientes}</b></div>
    <div class="small">Cortes contratados: <b>${totalContratados}</b></div>
    <div class="small">Cortes realizados: <b>${totalFeitos}</b></div>
    <div class="small">Receita confirmada: <b>${money(receitaConfirmada)}</b></div>
  `;
}

/* ---------- init ---------- */
render();
</script>
</body>
</html>

