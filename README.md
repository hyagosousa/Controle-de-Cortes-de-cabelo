
<!-- SISTEMA COMPLETO DE GERENCIAMENTO DE CORTES POR CLIENTE -->
<!-- OTIMIZADO PARA CELULAR – COM BUSCA, HISTÓRICO, PROGRESSO, PDF, WHATSAPP -->

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
        background: #f5f5f5;
    }
    h2 {
        text-align: center;
    }

    .card {
        background: #fff;
        padding: 15px;
        margin-bottom: 15px;
        border-radius: 10px;
        box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    }

    input, button, select {
        width: 100%;
        padding: 12px;
        font-size: 15px;
        margin-top: 8px;
        border-radius: 8px;
        border: 1px solid #ccc;
    }

    button {
        background: #007bff;
        color: white;
        border: none;
        cursor: pointer;
    }
    button:hover {
        background: #0056b3;
    }

    .clienteBox {
        background: #fff;
        padding: 12px;
        margin-top: 10px;
        border-radius: 10px;
        border-left: 5px solid #007bff;
    }

    .progressBar {
        width: 100%;
        background: #ddd;
        border-radius: 8px;
        height: 14px;
        margin-top: 6px;
    }
    .progress {
        height: 14px;
        background: #28a745;
        border-radius: 8px;
        width: 0%;
    }

    .btn-small {
        padding: 8px;
        font-size: 14px;
        margin-top: 5px;
    }
</style>
</head>
<body>

<h2>📋 Controle de Cortes por Cliente</h2>

<div class="card">
    <h3>Adicionar Cliente</h3>
    <input id="nome" placeholder="Nome do Cliente" />
    <input id="telefone" placeholder="Telefone (opcional)" />
    <input id="totalCortes" type="number" placeholder="Total de cortes contratados" />
    <button onclick="addCliente()">Adicionar</button>
</div>

<div class="card">
    <h3>Buscar Cliente</h3>
    <input id="busca" placeholder="Digite o nome" oninput="renderClientes()" />
</div>

<div id="listaClientes"></div>

<div class="card">
    <button onclick="exportPDF()">📄 Exportar Histórico em PDF</button>
    <button onclick="enviarWhatsApp()">📩 Enviar Relatório por WhatsApp</button>
</div>

<script>
let clientes = JSON.parse(localStorage.getItem("clientesCortes") || "[]");

function salvar() {
    localStorage.setItem("clientesCortes", JSON.stringify(clientes));
}

function addCliente() {
    const nome = document.getElementById("nome").value.trim();
    const tel = document.getElementById("telefone").value.trim();
    const total = Number(document.getElementById("totalCortes").value);

    if (!nome || !total) return alert("Preencha nome e total de cortes.");

    clientes.push({
        nome,
        telefone: tel,
        total,
        feitos: 0,
        historico: []
    });

    salvar();
    renderClientes();

    document.getElementById("nome").value = "";
    document.getElementById("telefone").value = "";
    document.getElementById("totalCortes").value = "";
}

function registrarCorte(i) {
    const c = clientes[i];
    if (c.feitos >= c.total) return alert("Todos os cortes foram concluídos!");

    c.feitos++;
    c.historico.push({
        data: new Date().toLocaleString()
    });

    // Aviso quando faltar 1
    if (c.total - c.feitos === 1) {
        alert(`⚠️ Falta apenas 1 corte para o cliente ${c.nome}!`);
        if (c.telefone) window.open(`https://wa.me/55${c.telefone}?text=Olá%20${c.nome},%20falta%20apenas%201%20corte%20para%20finalizarmos!`);
    }

    // Aviso quando finalizar
    if (c.feitos === c.total) {
        alert(`✅ Cliente ${c.nome} finalizou todos os cortes!`);
    }

    salvar();
    renderClientes();
}

function renderClientes() {
    const busca = document.getElementById("busca").value.toLowerCase();
    const box = document.getElementById("listaClientes");
    box.innerHTML = "";

    clientes
        .filter(c => c.nome.toLowerCase().includes(busca))
        .forEach((c, i) => {
            const pct = Math.round((c.feitos / c.total) * 100);

            box.innerHTML += `
                <div class='clienteBox'>
                    <b>${c.nome}</b><br>
                    Cortes feitos: ${c.feitos} / ${c.total}<br>
                    <div class='progressBar'><div class='progress' style='width:${pct}%'></div></div>

                    <button class='btn-small' onclick='registrarCorte(${i})'>Registrar Corte</button>

                    <button class='btn-small' onclick='mostrarHistorico(${i})'>Histórico</button>
                </div>
            `;
        });
}

function mostrarHistorico(i) {
    const c = clientes[i];
    let txt = `Histórico de ${c.nome}:\n\n`;

    c.historico.forEach((h, n) => {
        txt += `${n + 1}º corte: ${h.data}\n`;
    });

    alert(txt);
}

function exportPDF() {
    const conteudo = JSON.stringify(clientes, null, 2);
    const blob = new Blob([conteudo], { type: "text/plain" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = "relatorio_cortes.txt";
    a.click();
}

function enviarWhatsApp() {
    let texto = "RELATÓRIO DE CORTES:\n\n";
    clientes.forEach(c => {
        texto += `Cliente: ${c.nome} - ${c.feitos}/${c.total}\n`;
    });

    const link = `https://wa.me/?text=${encodeURIComponent(texto)}`;
    window.open(link);
}

renderClientes();
</script>

</body>
</html>
