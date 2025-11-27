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
        background: #0a0a0a;
        color: #e0e0e0;
    }
    h2, h3 {
        text-align: center;
        color: #4da3ff;
    }

    .card {
        background: #111;
        padding: 15px;
        margin-bottom: 15px;
        border-radius: 10px;
        box-shadow: 0 0 10px rgba(0,123,255,0.3);
        border: 1px solid #004c99;
    }

    input, button, select {
        width: 100%;
        padding: 12px;
        font-size: 15px;
        margin-top: 8px;
        border-radius: 8px;
        border: 1px solid #1a1a1a;
    }

    input {
        background: #0f0f0f;
        color: white;
    }

    button {
        background: #0066cc;
        color: white;
        border: none;
        cursor: pointer;
        transition: 0.2s;
    }
    button:hover {
        background: #0052a3;
    }

    .btn-confirmar {
        background: #0f9d58 !important;
        color: #fff !important;
    }

    .clienteBox {
        background: #111;
        padding: 12px;
        margin-top: 10px;
        border-radius: 10px;
        border-left: 5px solid #1e90ff;
        box-shadow: 0 0 8px rgba(30,144,255,0.4);
    }

    .progressBar {
        width: 100%;
        background: #333;
        border-radius: 8px;
        height: 14px;
        margin-top: 6px;
    }
    .progress {
        height: 14px;
        background: #1e90ff;
        border-radius: 8px;
        width: 0%;
        transition: width .3s;
    }

    .btn-small {
        padding: 8px;
        font-size: 14px;
        margin-top: 5px;
        background: #004c99;
        border: none;
        color: white;
        border-radius: 8px;
    }
    .btn-small:hover {
        background: #003d7a;
    }
</style>
</head>
<body>

<h2>📋 Controle de Cortes por Cliente</h2>

<div class="card">
    <h3>➕ Adicionar Cliente</h3>
    <input id="nome" placeholder="Nome do Cliente" />
    <input id="telefone" placeholder="Telefone (opcional)" />
    <input id="valorPlano" type="number" placeholder="Valor do Plano (R$)" />
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
    const nome = document.getElementById("nome").value.trim();
    const tel = document.getElementById("telefone").value.trim();
    const valor = Number(document.getElementById("valorPlano").value);
    const total = Number(document.getElementById("totalCortes").value);

    if (!nome || !total || !valor) return alert("Preencha nome, valor e total de cortes.");

    clientes.push({
        nome,
        telefone: tel,
        valor,
        total,
        feitos: 0,
        historico: []
    });

    salvar();
    renderClientes();

    document.getElementById("nome").value = "";
    document.getElementById("telefone").value = "";
    document.getElementById("valorPlano").value = "";
    document.getElementById("totalCortes").value = "";
}

function registrarCorte(i)(i){
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
    let texto = `Histórico de ${c.nome}:

`;
    c.historico.forEach((h,n)=> texto += `${n+1}º corte: ${h.data}
`);
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
    let texto = "RELATÓRIO DE CORTES:

";
    clientes.forEach(c=> texto += `Cliente: ${c.nome} - ${c.feitos}/${c.total}
`);
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

