const SHEET_URL = "/db"; 
        const GID = "116701254"; 
        
        // --- LOGIC UTAMA ---
        let order = {};
        const PAY_METHODS = Config.methods;

        document.addEventListener("DOMContentLoaded", () => {
            lucide.createIcons();
            const trx = new URLSearchParams(location.search).get('trx');
            if (trx) fetchOrder(trx);
            else showError("ID Transaksi tidak ditemukan.");
        });

        function fetchOrder(trxId) {
            const query = `select G, I, J, K, L, M, N where B = '${localStorage.getItem('uuid') || UUID}'`; 
            
            fetch(`${SHEET_URL}?&tq=${encodeURIComponent(query)}&gid=${GID}`)
                .then(r => r.text())
                .then(txt => {
                    const json = JSON.parse(txt.substring(txt.indexOf("{"), txt.lastIndexOf("}") + 1));
                    const row = (json.table.rows || []).find(r => r.c[3]?.v === trxId);

                    if (!row) return showError("Pesanan tidak ditemukan.");

                    order = {
                        id: trxId,
                        total: row.c[2]?.v || 0,
                        methodName: (row.c[4]?.v || 'BCA').trim(),
                        status: (row.c[6]?.v || 'PENDING').trim().toUpperCase(),
                        items: parseItems(row.c[0]?.v)
                    };

                    const mName = order.methodName.toLowerCase();
                    order.methodKey = Object.keys(PAY_METHODS).find(k => mName.includes(k)) || 'bca';

                    renderUI();
                })
                .catch(e => {
                    console.error(e);
                    showError("Gagal memuat data.");
                });
        }

        function parseItems(jsonStr) {
            if(!jsonStr) return [{title: "Produk", quantity: 1}];
            try {
                const parsed = typeof jsonStr === 'string' ? JSON.parse(jsonStr) : jsonStr;
                return Array.isArray(parsed) ? parsed : [parsed];
            } catch { return [{title: "Produk", quantity: 1}]; }
        }

        function renderUI() {
            // Render Header & Content State
            document.getElementById('loading-state').classList.add('hidden');
            document.getElementById('content-state').classList.remove('hidden');
            
            document.getElementById('header-order-id').innerText = order.id;
            document.getElementById('item-total').innerText = "Rp " + parseInt(order.total).toLocaleString('id-ID');
            
            // List Items
            const itemListHtml = order.items.map(i => {
                const titles = i.title || i.name || i.item || 'Produk';
                const qty = i.quantity || i.qty || 1;
                const img = i.image || 'https://via.placeholder.com/80?text=IMG';
                const variantHtml = i.varian ? `<p class="text-xs text-gray-500 mt-0.5">Varian: ${i.varian}</p>` : '';
                const priceHtml = i.price ? `<p class="text-xs font-semibold text-gray-600">Rp ${parseInt(i.price).toLocaleString('id-ID')}</p>` : '';

                return `
                <div class="flex gap-3 items-start border-b border-dashed border-gray-100 pb-3 last:border-0 last:pb-0">
                    <div class="w-16 h-16 flex-shrink-0 bg-gray-100 rounded-lg overflow-hidden border border-gray-200">
                        <img src="${img}" alt="${titles}" class="w-full h-full object-cover">
                    </div>
                    <div class="flex-1 min-w-0">
                        <h4 class="text-sm font-medium text-gray-800 line-clamp-2 leading-snug">${titles}</h4>
                        ${variantHtml}
                        <div class="flex justify-between items-end mt-1">
                            <span class="text-xs text-gray-500 bg-gray-100 px-2 py-0.5 rounded font-medium">x${qty}</span>
                            ${priceHtml}
                        </div>
                    </div>
                </div>`;
            }).join('');
            
            document.getElementById('item-list-container').innerHTML = itemListHtml;

            const totalQty = order.items.reduce((a,b) => a + parseInt(b.quantity||b.qty||1), 0);
            document.getElementById('item-qty').innerText = totalQty;

            // Render Status Badge
            const badge = document.getElementById('status-badge');
            let color = "yellow", icon = "clock", msg = order.status;
            
            if (['SUKSES','DONE','SELESAI','LUNAS'].some(s => order.status.includes(s))) { color = "green"; icon = "check-circle"; }
            else if (['CANCEL','GAGAL','BATAL','TOLAK'].some(s => order.status.includes(s))) { color = "red"; icon = "times-circle"; }
            else if (['PROCES','KEMAS','DIKIRIM','SENT','OTW'].some(s => order.status.includes(s))) { color = "blue"; icon = "box-open"; }

            badge.className = `text-xs font-medium px-2 py-1 rounded-full border flex items-center gap-1 bg-${color}-50 text-${color}-600 border-${color}-200`;
            badge.innerHTML = `<i class="fas fa-${icon}"></i> ${msg}`;

            // Render Payment Area
            const area = document.getElementById('payment-area');
            const conf = PAY_METHODS[order.methodKey];
            const theme = conf.color; 

            // LOGIKA RENDER BERDASARKAN STATUS ORDER & CONFIG METHOD

            // 1. Jika Status Order SUKSES
            if (color === 'green') {
                area.innerHTML = `
                    <div class="text-center py-8">
                        <div class="w-16 h-16 bg-green-100 rounded-full flex items-center justify-center mx-auto mb-4"><i class="fas fa-check text-3xl text-green-600"></i></div>
                        <h3 class="text-xl font-bold text-gray-900">Pembayaran Berhasil</h3>
                        <p class="text-sm text-gray-500 mt-2">Terima kasih! Pesanan Anda telah lunas.</p>
                        <a href="/" class="mt-6 inline-block px-6 py-2 bg-blue-600 text-white rounded-lg font-medium hover:bg-blue-700">Belanja Lagi</a>
                    </div>
                `;
            }
            // 2. Jika Status Order BATAL
            else if (color === 'red') {
                const msgCancel = Config.waMessages.cancel.replace('{{ID}}', order.id);
                area.innerHTML = `
                    <div class="text-center py-8">
                        <div class="w-16 h-16 bg-red-100 rounded-full flex items-center justify-center mx-auto mb-4"><i class="fas fa-times text-3xl text-red-600"></i></div>
                        <h3 class="text-xl font-bold text-gray-900">Pesanan Dibatalkan</h3>
                        <p class="text-sm text-gray-500 mt-2">Pesanan ini telah dibatalkan oleh sistem/admin.</p>
                        <div class="mt-6 flex flex-col md:flex-row justify-center gap-3">
                            <button onclick="contactAdmin('${msgCancel}')" class="w-full md:w-auto md:px-8 py-3 bg-gray-800 text-white rounded-lg font-bold"><i class="fab fa-whatsapp"></i> Silahkan Chat Admin</button>
                            <a href="/" class="w-full md:w-auto md:px-8 py-3 bg-blue-600 text-white rounded-lg font-bold text-center">Beli Lagi</a>
                        </div>
                    </div>
                `;
            }
            // 3. Jika Status Order DIPROSES
            else if (color === 'blue') {
                const msgCheck = Config.waMessages.check.replace('{{ID}}', order.id);
                area.innerHTML = `
                    <div class="text-center py-8">
                        <div class="w-16 h-16 bg-blue-100 rounded-full flex items-center justify-center mx-auto mb-4"><i class="fas fa-truck-fast text-3xl text-blue-600"></i></div>
                        <h3 class="text-xl font-bold text-gray-900">Pesanan Sedang Diproses</h3>
                        <p class="text-sm text-gray-500 mt-2">Selamat! Pesanan sedang dikemas atau dalam perjalanan.</p>
                        <button onclick="contactAdmin('${msgCheck}')" class="mt-6 w-full md:w-auto md:px-8 py-3 bg-green-500 text-white rounded-lg font-bold"><i class="fab fa-whatsapp"></i> Hubungi Admin</button>
                    </div>
                `;
            }
            // 4. Jika Status PENDING (Menunggu Pembayaran)
            else {
                // CEK STATUS CONFIG METODE (ON/OFF)
                if (conf.status === 'off') {
                    // TAMPILAN JIKA METODE OFF
                    const msgChange = Config.waMessages.change.replace('{{ID}}', order.id);
                    area.innerHTML = `
                        <div class="flex justify-between items-start mb-4 border-b border-gray-100 pb-3">
                            <div>
                                <h2 class="text-lg font-bold text-gray-900">${conf.name}</h2>
                                <p class="text-xs text-red-500 font-bold">Metode Tidak Tersedia</p>
                            </div>
                            <span class="text-gray-400 font-bold text-xl italic">${order.methodKey.toUpperCase()}</span>
                        </div>
                        <div class="bg-red-50 p-6 rounded-xl border border-red-100 text-center mb-6">
                            <i class="fas fa-ban text-4xl text-red-400 mb-3"></i>
                            <h3 class="font-bold text-red-800">Pembayaran Nonaktif</h3>
                            <p class="text-sm text-red-600 mt-2">Mohon maaf, metode pembayaran ini sedang gangguan atau dinonaktifkan.</p>
                            <p class="text-sm text-red-600 font-bold mt-1">Silahkan ganti pembayaran.</p>
                        </div>
                        <div class="text-center">
                            <button onclick="contactAdmin('${msgChange}')" class="w-full md:w-auto md:px-12 bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 rounded-xl shadow-lg flex items-center justify-center gap-2 mx-auto">
                                <i class="fab fa-whatsapp text-lg"></i> Hubungi Admin
                            </button>
                        </div>
                    `;
                } else {
                    // TAMPILAN NORMAL (METODE ON)
                    let detailsHTML = '';
                    if(conf.type === 'QRIS') {
                        detailsHTML = `<div class="flex justify-center mb-4"><img src="${conf.img}" class="w-40 h-40 border rounded-lg"></div>`;
                    } else if (conf.type === 'COD') {
                        detailsHTML = `<div class="bg-gray-50 p-4 rounded-lg text-center mb-4 border"><i class="fas fa-truck text-2xl text-gray-400 mb-2"></i><p class="text-sm">Bayar tunai saat kurir sampai.</p></div>`;
                    } else {
                        detailsHTML = `
                            <div class="bg-${theme}-50 p-4 rounded-xl border border-${theme}-100 mb-4 relative">
                                <div class="text-[10px] uppercase font-bold text-${theme}-600 mb-1">Nomor ${conf.type === 'VA' ? 'Virtual Account' : 'Rekening'}</div>
                                <div class="flex justify-between items-center">
                                    <span class="text-xl md:text-2xl font-mono font-bold text-gray-800">${conf.number}</span>
                                    <button onclick="copyText('${conf.number}')" class="text-${theme}-600 bg-white px-3 py-1 rounded shadow-sm text-sm font-medium">Salin</button>
                                </div>
                            </div>
                            <div class="bg-gray-50 p-3 rounded-lg text-xs text-gray-500 mb-4 border">
                                <p class="font-bold mb-1">Cara Bayar:</p>
                                <ol class="list-decimal pl-4 space-y-1">
                                    <li>Buka Aplikasi ${conf.name.split(' ')[0]} Mobile/ATM.</li>
                                    <li>Pilih menu Transfer / Bayar.</li>
                                    <li>Masukkan Nomor di atas.</li>
                                    <li>Konfirmasi & Simpan Bukti.</li>
                                </ol>
                            </div>
                        `;
                    }

                    area.innerHTML = `
                        <div class="flex justify-between items-start mb-4 border-b border-gray-100 pb-3">
                            <div>
                                <h2 class="text-lg font-bold text-gray-900">${conf.name}</h2>
                                <p class="text-xs text-gray-500">Selesaikan pembayaran sebelum kadaluarsa.</p>
                            </div>
                            <span class="text-${theme}-600 font-bold text-xl italic">${order.methodKey.toUpperCase()}</span>
                        </div>
                        ${detailsHTML}
                        <div class="text-center">
                            <button onclick="confirmWA()" class="w-full md:w-auto md:px-12 bg-green-500 hover:bg-green-600 text-white font-bold py-3 rounded-xl shadow-lg shadow-green-100 flex items-center justify-center gap-2 mx-auto">
                                <i class="fab fa-whatsapp text-lg"></i> Konfirmasi Pembayaran
                            </button>
                        </div>
                    `;
                }
            }
        }

        function copyText(txt) {
            navigator.clipboard.writeText(txt);
            const t = document.getElementById('toast');
            t.classList.remove('opacity-0'); setTimeout(() => t.classList.add('opacity-0'), 2000);
        }

        // --- NEW: confirmWA menggunakan Template Config & Replace ---
        function confirmWA() {
            const conf = PAY_METHODS[order.methodKey];
            const formattedTotal = "Rp " + parseInt(order.total).toLocaleString('id-ID');
            
            // Format List Item
            const itemsList = order.items.map(i => `- ${i.title||i.name||i.item||'Produk'} (x${i.qty||i.quantity||1})`).join('\n');
            const destination = (conf.type !== 'QRIS' && conf.type !== 'COD') ? `Tujuan: ${conf.number}` : '';

            // Replace Template
            let message = Config.waMessages.confirm
                .replace('{{ID}}', order.id)
                .replace('{{TOTAL}}', formattedTotal)
                .replace('{{METODE}}', conf.name)
                .replace('{{TUJUAN}}', destination)
                .replace('{{ITEM}}', itemsList);

            // Bersihkan baris kosong jika TUJUAN kosong
            if (!destination) {
                message = message.replace('\n\n\n', '\n\n'); 
            }

            window.open(`https://wa.me/${Config.waAdmin.no}?text=${encodeURIComponent(message)}`, '_blank');
        }

        // --- NEW: contactAdmin untuk tombol status lainnya ---
        function contactAdmin(msg) {
            window.open(`https://wa.me/${Config.waAdmin.no}?text=${encodeURIComponent(msg)}`, '_blank');
        }

        function showError(m) { 
            document.getElementById('loading-state').innerHTML = `<p class="text-red-500">${m}</p>`; 
        }
