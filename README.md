import datetime
import json
import os
from flask import Flask, jsonify, render_template_string, request

app = Flask(__name__)
DOSYA_ADI = "notlar_otantik.json"


def notlari_yukle():
  if os.path.exists(DOSYA_ADI):
    try:
      with open(DOSYA_ADI, "r", encoding="utf-8") as f:
        return json.load(f)
    except:
      return []
  return []


def notlari_kaydet(notlar):
  with open(DOSYA_ADI, "w", encoding="utf-8") as f:
    json.dump(notlar, f, ensure_ascii=False, indent=4)


HTML_SAYFA = """
<!DOCTYPE html>
<html lang="tr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>🌿 Otantik Hatıra Defteri</title>
    <style>
        body { background-color: #1e1b18; color: #f4ebd0; font-family: Georgia, serif; margin: 0; padding: 15px; }
        h1 { text-align: center; font-size: 22px; color: #f4ebd0; }
        .card { background-color: #2e2925; border: 1px solid #3a322a; border-radius: 8px; padding: 15px; margin-bottom: 12px; }
        .badge { background-color: #3d352e; color: #b5a48b; font-size: 11px; padding: 3px 8px; border-radius: 4px; }
        .title { font-size: 16px; font-weight: bold; margin-top: 8px; }
        .date { font-size: 11px; color: #b5a48b; margin-top: 4px; }
        input, textarea, select { width: 100%; background: #26221e; border: 1px solid #8c6d53; color: #f4ebd0; padding: 10px; border-radius: 6px; margin-top: 5px; box-sizing: border-box; font-family: Georgia; }
        button { background: #8c6d53; color: #f4ebd0; border: none; padding: 12px; border-radius: 6px; width: 100%; font-weight: bold; font-family: Georgia; margin-top: 10px; cursor: pointer; }
        button:hover { background: #a38164; }
        .form-box { background: #26221e; padding: 15px; border-radius: 8px; border: 1px solid #3a322a; margin-bottom: 20px; }
    </style>
</head>
<body>
    <h1>✒️ Otantik Not Defteri</h1>
    
    <div class="form-box">
        <h3>Yeni Sayfa Ekle</h3>
        <input type="text" id="baslik" placeholder="Başlık...">
        <select id="kategori">
            <option value="Günlük">Günlük</option>
            <option value="İş">İş</option>
            <option value="Görev">Görev</option>
            <option value="Fikir">Fikir</option>
            <option value="Önemli">Önemli</option>
        </select>
        <textarea id="icerik" rows="4" placeholder="Düşüncelerin..."></textarea>
        <button onclick="notEkle()">✨ Deftere Yaz</button>
    </div>

    <h3>📜 Kayıtlı Sayfalar</h3>
    <div id="notlarListesi"></div>

    <script>
        async function notlariGetir() {
            let res = await fetch('/api/notlar');
            let notlar = await res.json();
            let listeDiv = document.getElementById('notlarListesi');
            listeDiv.innerHTML = '';
            
            if(notlar.length === 0) {
                listeDiv.innerHTML = '<p style="color: #b5a48b; text-align: center;">Defter henüz boş...</p>';
                return;
            }

            notlar.reverse().forEach(n => {
                listeDiv.innerHTML += `
                    <div class="card">
                        <span class="badge">${n.kategori}</span>
                        <div class="title">${n.baslik}</div>
                        <div class="date">📅 ${n.tarih}</div>
                        <p style="margin-top:8px; font-size:13px; color:#dcd6c4;">${n.icerik}</p>
                        <button style="background:#5c3a35; padding:6px; font-size:11px;" onclick="notSil(${n.id})">🗑️ Sayfayı Yırt / Sil</button>
                    </div>
                `;
            });
        }

        async function notEkle() {
            let baslik = document.getElementById('baslik').value;
            let kategori = document.getElementById('kategori').value;
            let icerik = document.getElementById('icerik').value;

            if(!baslik || !icerik) {
                alert("Başlık ve içerik boş olamaz!");
                return;
            }

            await fetch('/api/not-ekle', {
                method: 'POST',
                headers: {'Content-Type': 'application/json'},
                body: JSON.stringify({baslik, kategori, icerik})
            });

            document.getElementById('baslik').value = '';
            document.getElementById('icerik').value = '';
            notlariGetir();
        }

        async function notSil(id) {
            if(confirm("Bu sayfayı silmek istediğine emin misin?")) {
                await fetch('/api/not-sil/' + id, {method: 'DELETE'});
                notlariGetir();
            }
        }

        notlariGetir();
    </script>
</body>
</html>
"""


@app.route("/")
def index():
  return render_template_string(HTML_SAYFA)


@app.route("/api/notlar", methods=["GET"])
def api_notlar():
  return jsonify(notlari_yukle())


@app.route("/api/not-ekle", methods=["POST"])
def api_not_ekle():
  data = request.json
  notlar = notlari_yukle()
  simdiki_zaman = datetime.datetime.now().strftime("%d.%m.%Y - %H:%M")

  yeni_not = {
      "id": len(notlar) + 1,
      "kategori": data.get("kategori", "Günlük"),
      "baslik": data.get("baslik"),
      "icerik": data.get("icerik"),
      "tarih": simdiki_zaman,
  }
  notlar.append(yeni_not)
  notlari_kaydet(notlar)
  return jsonify({"durum": "başarılı"})


@app.route("/api/not-sil/<int:not_id>", methods=["DELETE"])
def api_not_sil(not_id):
  notlar = notlari_yukle()
  notlar = [n for n in notlar if n["id"] != not_id]
  notlari_kaydet(notlar)
  return jsonify({"durum": "başarılı"})


if __name__ == "__main__":
  # 0.0.0.0 ile başlatıyoruz ki aynı Wi-Fi ağındaki telefondan erişilebilsin
  app.run(host="0.0.0.0", port=5000, debug=True)
