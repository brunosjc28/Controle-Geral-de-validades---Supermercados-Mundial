<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Gestão de Validades - Supermercado Mundial</title>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
    <!-- Biblioteca Html5-Qrcode para leitura por câmera -->
    <script src="https://unpkg.com/html5-qrcode" type="text/javascript"></script>
    <!-- Firebase SDKs (Banco de dados em tempo real) -->
    <script src="https://www.gstatic.com/firebasejs/8.10.1/firebase-app.js"></script>
    <script src="https://www.gstatic.com/firebasejs/8.10.1/firebase-database.js"></script>
    <style>
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; background-color: #f8fafc; color: #1e293b; margin: 0; padding: 0; }
        header { background-color: #047857; color: white; padding: 16px; display: flex; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 10px; }
        .container { max-width: 1200px; margin: 0 auto; padding: 16px; }
        .kpi-grid { display: grid; grid-template-columns: repeat(2, 1fr); gap: 12px; margin-bottom: 20px; }
        @media(min-width: 768px) { .kpi-grid { grid-template-columns: repeat(4, 1fr); } }
        .kpi-card { background: white; padding: 16px; border-radius: 12px; border: 1px solid #e2e8f0; box-shadow: 0 1px 3px rgba(0,0,0,0.05); }
        .kpi-title { font-size: 11px; font-weight: 700; text-transform: uppercase; color: #64748b; margin: 0; }
        .kpi-value { font-size: 24px; font-weight: bold; margin-top: 4px; }
        .card { background: white; border-radius: 12px; border: 1px solid #e2e8f0; padding: 16px; margin-bottom: 20px; box-shadow: 0 1px 3px rgba(0,0,0,0.05); }
        .btn { background-color: #059669; color: white; border: none; padding: 10px 16px; border-radius: 8px; font-size: 14px; font-weight: 600; cursor: pointer; display: inline-flex; align-items: center; gap: 8px; }
        .btn-dark { background-color: #1e293b; }
        .btn-pdf { background-color: #065f46; font-size: 12px; padding: 8px 12px; }
        table { width: 100%; border-collapse: collapse; margin-top: 10px; font-size: 13px; }
        th, td { padding: 12px; text-align: left; border-bottom: 1px solid #f1f5f9; }
        th { background-color: #f8fafc; color: #64748b; font-size: 11px; text-transform: uppercase; }
        .badge { padding: 4px 8px; border-radius: 20px; font-size: 11px; font-weight: 600; display: inline-block; }
        .badge-vencido { background-color: #ffe4e6; color: #9f1239; }
        .badge-hoje { background-color: #ffedd5; color: #c2410c; }
        .badge-atencao-vermelho { background-color: #fee2e2; color: #dc2626; font-weight: bold; border: 1px solid #fca5a5; }
        .badge-normal { background-color: #ecfdf5; color: #047857; }
        /* Modal */
        .modal { display: none; position: fixed; inset: 0; background: rgba(15, 23, 42, 0.6); align-items: center; justify-content: center; padding: 16px; z-index: 100; }
        .modal.flex { display: flex; }
        .modal-content { background: white; border-radius: 16px; width: 100%; max-width: 500px; padding: 20px; max-height: 90vh; overflow-y: auto; box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1); }
        .form-group { margin-bottom: 14px; }
        .form-group label { display: block; font-size: 11px; font-weight: 700; text-transform: uppercase; color: #475569; margin-bottom: 6px; }
        .form-control { width: 100%; padding: 10px; border: 1px solid #cbd5e1; border-radius: 8px; font-size: 14px; box-sizing: border-box; }
        .row { display: flex; gap: 12px; }
        .row > div { flex: 1; }
        #reader { width: 100%; max-width: 300px; margin: 10px auto; border-radius: 8px; overflow: hidden; }
    </style>
</head>
<body>

    <header>
        <div>
            <h1 style="margin:0; font-size: 18px;">Controle Geral de Validades & Lotes</h1>
            <p style="margin:2px 0 0 0; font-size: 11px; color: #a7f3d0;">Supermercado Mundial — Lojas & Depósito (Online)</p>
        </div>
        <button onclick="gerarRelatorioPDF()" class="btn btn-pdf">
            <i class="fa-solid fa-file-pdf"></i> Relatório PDF (Críticos / ≤ 5 dias)
        </button>
    </header>

    <div class="container">

        <!-- KPIs -->
        <div class="kpi-grid">
            <div class="kpi-card">
                <p class="kpi-title">Total Monitorado</p>
                <div id="stat-total" class="kpi-value" style="color: #1e293b;">0</div>
            </div>
            <div class="kpi-card">
                <p class="kpi-title" style="color: #dc2626;">Atenção (≤ 5 dias)</p>
                <div id="stat-warning" class="kpi-value" style="color: #dc2626;">0</div>
            </div>
            <div class="kpi-card">
                <p class="kpi-title" style="color: #ea580c;">Vencem Hoje</p>
                <div id="stat-today" class="kpi-value" style="color: #ea580c;">0</div>
            </div>
            <div class="kpi-card">
                <p class="kpi-title" style="color: #e11d48;">Vencidos</p>
                <div id="stat-expired" class="kpi-value" style="color: #e11d48;">0</div>
            </div>
        </div>

        <!-- Ação de Cadastro -->
        <div class="card" style="display: flex; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 12px;">
            <div>
                <h3 style="margin:0; font-size: 15px; color: #334155;">Produtos e Lotes na Nuvem</h3>
                <p style="margin:2px 0 0 0; font-size: 12px; color: #64748b;">Sincronizado em tempo real</p>
            </div>
            <button onclick="abrirModalNovo()" class="btn" style="width: 100%; max-width: 280px; justify-content: center;">
                <i class="fa-solid fa-plus-circle"></i> Registrar Lote / Validade
            </button>
        </div>

        <!-- Tabela -->
        <div class="card" style="overflow-x: auto; padding: 0;">
            <table>
                <thead>
                    <tr>
                        <th>Produto / EAN</th>
                        <th>Setor</th>
                        <th>Lote</th>
                        <th>Validade</th>
                        <th>Status</th>
                        <th>Ação de Retirada</th>
                        <th style="text-align: right;">Ação</th>
                    </tr>
                </thead>
                <tbody id="tabela-produtos">
                    <tr><td colspan="7" style="text-align:center; color:#64748b; padding:20px;">Carregando dados da nuvem...</td></tr>
                </tbody>
            </table>
        </div>
    </div>

    <!-- Modal -->
    <div id="modal-produto" class="modal">
        <div class="modal-content">
            <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 16px;">
                <h3 style="margin:0; font-size: 16px;">Registrar Lote e Validade</h3>
                <button onclick="fecharModal()" style="background:none; border:none; font-size: 18px; cursor:pointer;"><i class="fa-solid fa-xmark"></i></button>
            </div>

            <!-- Câmera -->
            <div style="background: #f8fafc; padding: 12px; border-radius: 8px; border: 1px solid #e2e8f0; text-align: center; margin-bottom: 16px;">
                <button type="button" onclick="toggleCamera()" class="btn btn-dark" style="font-size: 12px; padding: 8px 12px;">
                    <i class="fa-solid fa-camera"></i> <span id="btn-texto-camera">Escanear Código de Barras</span>
                </button>
                <div id="reader" style="display: none;"></div>
            </div>

            <form id="form-produto" onsubmit="salvarProduto(event)">
                <div class="form-group">
                    <label>Código de Barras (EAN)</label>
                    <input type="text" id="input-ean" class="form-control" placeholder="Lido pela câmera ou digitado" oninput="verificarMemoriaEAN(this.value)">
                </div>
                <div class="form-group">
                    <label>Nome do Produto</label>
                    <input type="text" id="input-nome" required class="form-control" placeholder="Ex: Pão de Forma Integral 500g">
                </div>
                <div class="row">
                    <div class="form-group">
                        <label>Setor</label>
                        <select id="input-setor" class="form-control">
                            <option value="Padaria">Padaria</option>
                            <option value="Laticínios">Laticínios</option>
                            <option value="Açougue">Açougue</option>
                            <option value="Mercearia">Mercearia</option>
                            <option value="Frios e Embutidos">Frios e Embutidos</option>
                        </select>
                    </div>
                    <div class="form-group">
                        <label>Nº do Lote</label>
                        <input type="text" id="input-lote" class="form-control" placeholder="Ex: L-042" style="text-transform: uppercase;">
                    </div>
                </div>
                <div class="form-group">
                    <label>Data de Validade</label>
                    <input type="date" id="input-validade" required class="form-control">
                </div>
                <div style="display: flex; justify-content: flex-end; gap: 8px; border-top: 1px solid #f1f5f9; padding-top: 12px; margin-top: 16px;">
                    <button type="button" onclick="fecharModal()" style="background:#f1f5f9; color:#475569; border:none; padding:10px 16px; border-radius:8px; font-weight:600; cursor:pointer;">Cancelar</button>
                    <button type="submit" class="btn">Salvar Registro</button>
                </div>
            </form>
        </div>
    </div>

    <!-- Script de Integração com Firebase -->
    <script>
        // URL oficial do seu banco de dados Firebase
        const firebaseConfig = {
            databaseURL: "https://validade-mundial-default-rtdb.firebaseio.com/"
        };
        firebase.initializeApp(firebaseConfig);
        const db = firebase.database();

        const hoje = new Date();
        let html5QrCode = null;
        let cameraLigada = false;
        let produtos = [];
        let catalogoEAN = {};

        function formatarData(dataStr) {
            if (!dataStr) return '';
            const partes = dataStr.split('-');
            if (partes.length !== 3) return dataStr;
            return `${partes[2]}/${partes[1]}/${partes[0]}`;
        }

        // Ouvir alterações no Banco de Dados em Tempo Real
        db.ref('produtos').on('value', (snapshot) => {
            const data = snapshot.val();
            produtos = [];
            if (data) {
                Object.keys(data).forEach(key => {
                    produtos.push({ id: key, ...data[key] });
                });
            }
            renderizarTabela();
        }, (error) => {
            console.error("Erro ao ler dados:", error);
            document.getElementById('tabela-produtos').innerHTML = `<tr><td colspan="7" style="text-align:center; color:#dc2626; padding:20px;">Erro de permissão no Firebase. Verifique se as Regras (Rules) estão definidas como true.</td></tr>`;
        });

        // Ouvir catálogo de EANs
        db.ref('catalogoEAN').on('value', (snapshot) => {
            const data = snapshot.val();
            if (data) {
                catalogoEAN = data;
            }
        });

        function verificarMemoriaEAN(ean) {
            const eanLimpo = ean.trim();
            if (catalogoEAN[eanLimpo]) {
                document.getElementById('input-nome').value = catalogoEAN[eanLimpo].nome;
                document.getElementById('input-setor').value = catalogoEAN[eanLimpo].setor;
            }
        }

        function calcularStatus(dataValidadeStr) {
            const dataVal = new Date(dataValidadeStr + 'T00:00:00');
            const dataHj = new Date(hoje.getFullYear(), hoje.getMonth(), hoje.getDate());
            const diffDias = Math.ceil((dataVal - dataHj) / (1000 * 60 * 60 * 24));

            if (diffDias < 0) {
                return { tipo: 'vencido', texto: 'Vencido ❌', classe: 'badge-vencido', acao: 'Retirar (Perda)' };
            } else if (diffDias === 0) {
                return { tipo: 'hoje', texto: 'Vence Hoje ⚠️', classe: 'badge-hoje', acao: 'Queima (50% Off)' };
            } else if (diffDias <= 5) {
                return { tipo: 'atencao', texto: `Vence em ${diffDias}d 🚨`, classe: 'badge-atencao-vermelho', acao: 'Retirar p/ Retaguarda / Desconto' };
            } else {
                return { tipo: 'normal', texto: `No Prazo (${diffDias}d)`, classe: 'badge-normal', acao: 'Exposição Normal' };
            }
        }

        function renderizarTabela() {
            const tbody = document.getElementById('tabela-produtos');
            if (!tbody) return;
            tbody.innerHTML = '';
            
            if (produtos.length === 0) {
                tbody.innerHTML = `<tr><td colspan="7" style="text-align:center; color:#64748b; padding:20px;">Nenhum produto cadastrado na nuvem.</td></tr>`;
                document.getElementById('stat-total').innerText = '0';
                document.getElementById('stat-warning').innerText = '0';
                document.getElementById('stat-today').innerText = '0';
                document.getElementById('stat-expired').innerText = '0';
                return;
            }

            let total = 0, warning = 0, today = 0, expired = 0;

            produtos.sort((a, b) => new Date(a.validade) - new Date(b.validade));

            produtos.forEach(p => {
                total++;
                const status = calcularStatus(p.validade);
                if (status.tipo === 'atencao') warning++;
                if (status.tipo === 'hoje') today++;
                if (status.tipo === 'vencido') expired++;

                tbody.innerHTML += `
                    <tr>
                        <td><strong>${p.nome}</strong><br><small style="color:#64748b;">EAN: ${p.ean || 'N/D'}</small></td>
                        <td><span style="background:#f1f5f9; padding:4px 8px; border-radius:6px; font-size:12px;">${p.setor}</span></td>
                        <td style="font-family:monospace; font-weight:600; color:#334155;">${p.lote || 'S/ Lote'}</td>
                        <td>${formatarData(p.validade)}</td>
                        <td><span class="badge ${status.classe}">${status.texto}</span></td>
                        <td style="font-size:12px; color:#475569;"><i class="fa-solid fa-tag" style="color:#dc2626;"></i> ${status.acao}</td>
                        <td style="text-align: right;">
                            <button onclick="removerProduto('${p.id}')" style="background:none; border:none; color:#94a3b8; cursor:pointer; padding:6px;"><i class="fa-solid fa-trash-can"></i></button>
                        </td>
                    </tr>
                `;
            });

            document.getElementById('stat-total').innerText = produtos.length;
            document.getElementById('stat-warning').innerText = warning;
            document.getElementById('stat-today').innerText = today;
            document.getElementById('stat-expired').innerText = expired;
        }

        function abrirModalNovo() {
            const form = document.getElementById('form-produto');
            if (form) form.reset();
            const modal = document.getElementById('modal-produto');
            if (modal) modal.classList.add('flex');
        }

        function fecharModal() {
            pararCamera();
            const modal = document.getElementById('modal-produto');
            if (modal) modal.classList.remove('flex');
        }

        async function toggleCamera() {
            const readerDiv = document.getElementById('reader');
            const btnTexto = document.getElementById('btn-texto-camera');

            if (!cameraLigada) {
                if (readerDiv) readerDiv.style.display = 'block';
                if (btnTexto) btnTexto.innerText = "Parar Câmera";
                cameraLigada = true;

                try {
                    html5QrCode = new Html5Qrcode("reader");
                    await html5QrCode.start(
                        { facingMode: "environment" },
                        { fps: 10, qrbox: { width: 250, height: 100 } },
                        (decodedText) => {
                            const eanInput = document.getElementById('input-ean');
                            if (eanInput) {
                                eanInput.value = decodedText;
                                verificarMemoriaEAN(decodedText);
                            }
                            pararCamera();
                        },
                        (err) => {}
                    );
                } catch (err) {
                    alert("Aviso: O navegador pode bloquear a câmera em ambiente estático local. Se necessário, digite o código EAN.");
                    pararCamera();
                }
            } else {
                pararCamera();
            }
        }

        function pararCamera() {
            if (html5QrCode && cameraLigada) {
                html5QrCode.stop().then(() => {
                    cameraLigada = false;
                    const r = document.getElementById('reader');
                    const b = document.getElementById('btn-texto-camera');
                    if (r) r.style.display = 'none';
                    if (b) b.innerText = "Escanear Código de Barras";
                }).catch(() => {
                    cameraLigada = false;
                    const r = document.getElementById('reader');
                    const b = document.getElementById('btn-texto-camera');
                    if (r) r.style.display = 'none';
                    if (b) b.innerText = "Escanear Código de Barras";
                });
            } else {
                cameraLigada = false;
                const r = document.getElementById('reader');
                const b = document.getElementById('btn-texto-camera');
                if (r) r.style.display = 'none';
                if (b) b.innerText = "Escanear Código de Barras";
            }
        }

        function salvarProduto(event) {
            event.preventDefault();
            const eanVal = document.getElementById('input-ean').value.trim() || 'SEM-EAN';
            const nomeVal = document.getElementById('input-nome').value.trim();
            const setorVal = document.getElementById('input-setor').value;
            const loteVal = document.getElementById('input-lote').value.trim().toUpperCase() || 'S/ LOTE';
            const validadeVal = document.getElementById('input-validade').value;

            if (eanVal !== 'SEM-EAN') {
                db.ref('catalogoEAN/' + eanVal).set({ nome: nomeVal, setor: setorVal });
            }

            // Salvar novo registro na nuvem
            db.ref('produtos').push({
                ean: eanVal,
                nome: nomeVal,
                setor: setorVal,
                lote: loteVal,
                validade: validadeVal
            }).then(() => {
                fecharModal();
            }).catch((err) => {
                alert("Erro ao salvar na nuvem: Verifique se as Regras (Rules) do Firebase estão como true.");
            });
        }

        function removerProduto(id) {
            if (confirm("Confirma a baixa deste item da nuvem?")) {
                db.ref('produtos/' + id).remove();
            }
        }

        function gerarRelatorioPDF() {
            const criticos = produtos.filter(p => {
                const status = calcularStatus(p.validade);
                return status.tipo === 'vencido' || status.tipo === 'hoje' || status.tipo === 'atencao';
            });

            if (criticos.length === 0) {
                alert("Nenhum produto crítico (≤ 5 dias ou vencido) para gerar relatório!");
                return;
            }

            const janelaImpressao = window.open('', '_print', 'width=800,height=600');
            
            let linhasTabela = '';
            criticos.forEach(p => {
                const status = calcularStatus(p.validade);
                linhasTabela += `
                    <tr>
                        <td><b>${p.nome}</b><br><small>EAN: ${p.ean || 'N/D'}</small></td>
                        <td>${p.setor}</td>
                        <td><b>${p.lote || 'S/ Lote'}</b></td>
                        <td>${formatarData(p.validade)}</td>
                        <td style="color: #dc2626; font-weight: bold;">${status.texto} — <i>${status.acao}</i></td>
                    </tr>
                `;
            });

            let htmlConteudo = `
                <html>
                <head>
                    <title>Relatório de Validades e Lotes - Ação de Retirada (≤ 5 Dias)</title>
                    <style>
                        body { font-family: Arial, sans-serif; padding: 20px; color: #333; }
                        h2 { color: #047857; margin-bottom: 5px; }
                        p { font-size: 12px; color: #666; margin-bottom: 20px; }
                        table { width: 100%; border-collapse: collapse; margin-top: 10px; }
                        th, td { border: 1px solid #ddd; padding: 10px; text-align: left; font-size: 12px; }
                        th { background-color: #f3f4f6; color: #374151; }
                    </style>
                </head>
                <body>
                    <h2>Supermercado Mundial - Lista de Retirada por Lote e Validade</h2>
                    <p>Itens críticos (vencidos, vencem hoje ou na janela de 5 dias) - Emitido em: ${new Date().toLocaleDateString('pt-BR')} às ${new Date().toLocaleTimeString('pt-BR')}</p>
                    <table>
                        <thead>
                            <tr>
                                <th>Produto</th>
                                <th>Setor</th>
                                <th>Lote</th>
                                <th>Validade</th>
                                <th>Status / Ação Recomendada</th>
                            </tr>
                        </thead>
                        <tbody>
                            ${linhasTabela}
                        </tbody>
                    </table>
                    <br><br>
                    <p>Assinatura do Encarregado: _____________________________________</p>
                    <script>
                        window.onload = function() { window.print(); }
                    <\/script>
                </body>
                </html>
            `;

            janelaImpressao.document.write(htmlConteudo);
            janelaImpressao.document.close();
        }
    </script>
</body>
</html>
