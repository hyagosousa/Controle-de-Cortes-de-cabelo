<!-- SISTEMA COMPLETO DE GERENCIAMENTO DE CORTES POR CLIENTE -->
<!-- MODO ESCURO PREMIUM + DASHBOARD FINAL -->

<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>Controle de Cortes</title>
<style>
    body {
        font-family: Arial, sans-serif;
        margin: 0; padding: 15px;
        background: #0d0d0d;
        color: #f2f2f2;
    }
    h2 { text-align: center; color: #00c8ff; }
    h3 { color: #0aff9d; }

    .card {
        background: #1a1a1a;
        padding: 15px;
        margin-bottom: 15px;
        border-radius: 12px;
        border: 1px solid #222;
        box-shadow: 0 0 10px rgba(0,255,255,0.1);
    }

    input, button, select {
        width: 100%; padding: 12px; font-size: 15px;
        margin-top: 8px;
        border-radius: 10px;
        background: #111;
        color: white;
        border: 1px solid #333;
    }

    button {
        background: linear-gradient(90deg,#00c8ff,#0084ff);
        border: none;
        cursor: pointer;
        font-weight: bold;
    }

    button:hover {
        opacity: 0.85;
    }

    .clienteBox {
        background: #141414;
        padding: 12px;
        margin-top: 10px;
        border-radius: 12px;
        border-left: 5px solid #00c8ff;
        box-shadow: 0 0 10px rgba(0,200,255,0.15);
    }

    .progressBar {
        width: 100%; background: #333;
        border-radius: 8px; height: 14px;
        margin-top: 6px;
    }
    .progress {
        height: 14px;
        background: linear-gradient(90deg,#0aff9d,#00ff6a);
        border-radius: 8px;
        width: 0%;
        transition: width .3s;
    }

    .btn-small {
        padding: 9px;
        font-size: 14px;
        margin-top: 6px;
        background: #222;
        border: 1px solid #444;
    }

    .dashboard {
        background: #111;
        padding: 15px;
        border-radius: 12px;
        margin-top: 40px;
        border: 1px solid #222;
        box-shadow: 0 0 12px rgba(0,255,180,0.08);
    }

</style>
</head>
<body>

<h2>📋 Controle de Cortes por Cliente</h2>

<div class="card">
    <h3>➕ Adicionar Cliente</h3>
    <input id="nome" placeholder="Nome do Cliente" />
    <input id="telefone" placeholder="Telefone (opcional)" />
    <input id="totalCortes" type="number" placeholder="Total de cortes contratados" />
    <button onclick="addCliente()">Adicionar</button>
</div>

<div class="card">
    <h3>🔍 Buscar Cliente</h3>
    <input id="busca" placeholder="Digite o nome" oninput="renderClientes()" />
</div>

<div id="listaClientes"></div>

<div class="card">
    <button onclick="exportPDF()">📄 Exportar Histórico em PDF</button>
    <button onclick="enviarWhatsApp()">📩 Enviar Relatório por WhatsApp</button>
</div>

<div class="dashboard">
    <h3>📊 Dashboard Geral</h3>
    <div id="dashResumo"></div>
</div>

<script>
let clientes = JSON.parse(localStorage.getItem("clientesCortes") || "[]");

function salvar(){ localStorage.setItem("clientesCortes", JSON.stringify(clientes)); }

function addCliente() {
    const nome = nome.value.trim();
    const tel = telefone.value.trim();
    const total = Number(totalCortes.value);

    if (!nome || !total) return alert("Preencha nome e total de cortes.");

    clientes.push({ nome, telefone: tel, total, feitos: 0, historico: [] });
    salvar();
    renderClientes();
    nome.value = telefone.value = totalCortes.value = "";
}

function registrarCorte(i){
    const c = clientes[i];
    if(c.feitos >= c.total) return alert("Este cliente já finalizou o plano.");

    c.feitos++;
    c.historico.push({ data: new Date().toLocaleString() });

    // ENVIAR MENSAGEM SOMENTE QUANDO FINALIZAR TODOS
    if(c.feitos === c.total && c.telefone){
        window.open(`https://wa.me/55${c.telefone}?text=Olá%20${c.nome},%20você%20acaba%20de%20concluir%20todos%20os%20cortes%20do%20seu%20plano!`);
    }

    salvar();
    renderClientes();
}

function renderClientes(){
    const buscaTxt = busca.value.toLowerCase();
    listaClientes.innerHTML = "";

    clientes.filter(c => c.nome.toLowerCase().includes(buscaTxt)).forEach((c,i)=>{
        const pct = Math.round((c.feitos / c.total) * 100);
        listaClientes.innerHTML += `
        <div class='clienteBox'>
            <b>${c.nome}</b><br>
            Cortes feitos: ${c.feitos}/${c.total}<br>
            <div class='progressBar'><div class='progress' style='width:${pct}%'></div></div>
            <button class='btn-small' onclick='registrarCorte(${i})'>Registrar Corte</button>
            <button class='btn-small' onclick='mostrarHistorico(${i})'>Histórico</button>
        </div>`;
    });

    atualizarDashboard();
}

function mostrarHistorico(i){
    const c = clientes[i];
    let texto = `Histórico de ${c.nome}:\n\n`;
    c.historico.forEach((h,n)=> texto += `${n+1}º corte: ${h.data}\n`);
    alert(texto);
}

function exportPDF(){
    const conteudo = JSON.stringify(clientes,null,2);
    const blob = new Blob([conteudo],{type:"text/plain"});
    const a = document.createElement("a");
    a.href = URL.createObjectURL(blob);
    a.download = "relatorio_cortes.txt";
    a.click();
}

function enviarWhatsApp(){
    let texto = "RELATÓRIO DE CORTES:\n\n";
    clientes.forEach(c=> texto += `Cliente: ${c.nome} - ${c.feitos}/${c.total}\n`);
    window.open("https://wa.me/?text=" + encodeURIComponent(texto));
}

function atualizarDashboard(){
    let totalClientes = clientes.length;
    let totalCortes = clientes.reduce((s,c)=> s + c.total, 0);
    let totalFeitos = clientes.reduce((s,c)=> s + c.feitos, 0);

    dashResumo.innerHTML = `
        <p>Total de clientes: <b>${totalClientes}</b></p>
        <p>Cortes totais contratados: <b>${totalCortes}</b></p>
        <p>Cortes já realizados: <b>${totalFeitos}</b></p>
        <p>Progresso geral: <b>${Math.round((totalFeitos/totalCortes)*100)||0}%</b></p>
    `;
}

renderClientes();
</script>

</body></html>
