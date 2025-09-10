---
jupyter:
  colab:
    toc_visible: true
  kernelspec:
    display_name: Python 3
    name: python3
  language_info:
    name: python
  nbformat: 4
  nbformat_minor: 0
---

::: {.cell .markdown id="jAGryMJp9-XB"}
**1. Crawling pta.trunojoyo.ac.id**
:::

::: {.cell .markdown id="e4PfAbd2-XkA"}
**Library**
:::

::: {.cell .code id="FECVHsej9qyT"}
``` python
import requests
from bs4 import BeautifulSoup
import pandas as pd
import re, sys, time
```
:::

::: {.cell .markdown id="WPDTCIRm_3po"}
**Base url**
:::

::: {.cell .code id="arwQEhSF-kVS"}
``` python
Base_Url = "https://pta.trunojoyo.ac.id/c_search/byprod"
```
:::

::: {.cell .markdown id="6rUAqcsdA698"}
**Function**
:::

::: {.cell .code id="PwBdcpGgA2d8"}
``` python
def get_max_page(prodi_id):
    url = f"{Base_Url}/{prodi_id}/1"
    r = requests.get(url)
    soup = BeautifulSoup(r.content, "html.parser")

    # Cari tombol >> (last page)
    last_page = soup.select_one('ol.pagination a:contains("»")')
    if last_page and "href" in last_page.attrs:
        href = last_page["href"]
        # Pecah URL -> ambil angka terakhir
        max_page = int(href.split("/")[-1])
        return max_page

    # fallback kalau pagination tidak ada
    return 1
```
:::

::: {.cell .code colab="{\"base_uri\":\"https://localhost:8080/\"}" id="QfXr7oVTBFkc" outputId="5b784cec-496d-4f79-c2cd-54499f367be9"}
``` python
print(get_max_page(10))
```

::: {.output .stream .stdout}
    172
:::
:::

::: {.cell .code id="cIO0fG_bDJwk"}
``` python
def print_progress(prodi_id, prodi, current_page, total_pages):
    percent = (current_page / total_pages) * 100
    bar_length = 20
    filled_length = int(bar_length * current_page // total_pages)
    bar = '█' * filled_length + '-' * (bar_length - filled_length)
    sys.stdout.write(f'\r[{prodi_id}] {prodi} - Page {current_page}/{total_pages} [{bar}] {percent:.2f}%')
    sys.stdout.flush()
    if current_page == total_pages:
        sys.stdout.write('\n')
```
:::

::: {.cell .markdown id="Xh052jG7A8Dr"}
**Function All Data**
:::

::: {.cell .code id="f7PDQWAUA5C-"}
``` python
def pta_all():
    start_time = time.time()

    data = {
        "id": [],
        "penulis": [],
        "judul": [],
        "abstrak_id": [],
        "abstrak_en": [],
        "pembimbing_pertama": [],
        "pembimbing_kedua": [],
        "prodi": []
    }

    total_prodi = 1
    total_pages = 0
    max_pages_dict = {}

    # hitung total halaman (untuk tiap prodi)
    for i in range(1, total_prodi + 1):
        max_page = get_max_page(i)
        max_pages_dict[i] = max_page
        total_pages += max_page

    for i in range(1, total_prodi + 1):
        max_page = max_pages_dict[i]
        for j in range(1, max_page + 1):
            url = f"{Base_Url}/{i}/{j}"
            r = requests.get(url)
            soup = BeautifulSoup(r.content, "html.parser")
            jurnals = soup.select('li[data-cat="#luxury"]')

            isii = soup.select_one('div#begin')
            if not isii:
                continue
            prodi_full = isii.select_one('h2').text.strip()
            prodi = prodi_full.replace("Journal Jurusan ", "")

            for jurnal in jurnals:
                link_keluar = jurnal.select_one('a.gray.button')['href']

                # ambil ID dari link PTA (angka terakhir di URL)
                id_match = re.search(r"/detail/(\d+)", link_keluar)
                pta_id = id_match.group(1) if id_match else None

                response = requests.get(link_keluar)
                soup1 = BeautifulSoup(response.content, "html.parser")
                isi = soup1.select_one('div#content_journal')

                judul = isi.select_one('a.title').text.strip()
                penulis = isi.select_one('span:contains("Penulis")').text.split(' : ')[1]
                pembimbing_pertama = isi.select_one('span:contains("Dosen Pembimbing I")').text.split(' : ')[1]
                pembimbing_kedua = isi.select_one('span:contains("Dosen Pembimbing II")').text.split(' :')[1]

                paragraf = isi.select('p[align="justify"]')
                abstrak_id = paragraf[0].get_text(strip=True) if len(paragraf) > 0 else "N/A"
                abstrak_en = paragraf[1].get_text(strip=True) if len(paragraf) > 1 else "N/A"

                data["id"].append(pta_id)
                data["penulis"].append(penulis)
                data["judul"].append(judul)
                data["abstrak_id"].append(abstrak_id)
                data["abstrak_en"].append(abstrak_en)
                data["pembimbing_pertama"].append(pembimbing_pertama)
                data["pembimbing_kedua"].append(pembimbing_kedua)
                data["prodi"].append(prodi)

            # update progress bar per prodi
            print_progress(i, prodi, j, max_page)

        sys.stdout.write("\n")  # pindah baris setelah 1 prodi selesai

    # simpan ke CSV
    df = pd.DataFrame(data)
    df.to_csv("pta_all.csv", index=False, encoding="utf-8-sig")

    # hitung durasi
    end_time = time.time()
    elapsed = int(end_time - start_time)
    jam, sisa = divmod(elapsed, 3600)
    menit, detik = divmod(sisa, 60)

    # summary
    print("\n✅ Seluruh data berhasil dikumpulkan!")
    print(f"📈 Total entri: {len(df)}")
    print(f"⏱️ Waktu eksekusi: {jam} jam {menit} menit {detik} detik")

    return df
```
:::

::: {.cell .code colab="{\"base_uri\":\"https://localhost:8080/\",\"height\":1000}" id="Lr96bpeqEZqB" outputId="c4820c5a-efe0-47db-e852-ef03b787d1be"}
``` python
pta_all()
```

::: {.output .stream .stderr}
    /usr/local/lib/python3.12/dist-packages/soupsieve/css_parser.py:876: FutureWarning: The pseudo class ':contains' is deprecated, ':-soup-contains' should be used moving forward.
      warnings.warn(  # noqa: B028
:::

::: {.output .stream .stdout}
    [1] Ilmu Hukum - Page 284/284 [████████████████████] 100.00%


    ✅ Seluruh data berhasil dikumpulkan!
    📈 Total entri: 1417
    ⏱️ Waktu eksekusi: 3 jam 3 menit 35 detik
:::

::: {.output .execute_result execution_count="22"}
``` json
{"summary":"{\n  \"name\": \"pta_all()\",\n  \"rows\": 1417,\n  \"fields\": [\n    {\n      \"column\": \"id\",\n      \"properties\": {\n        \"dtype\": \"string\",\n        \"num_unique_values\": 1417,\n        \"samples\": [\n          \"090111100139\",\n          \"120111100276\",\n          \"130111100263\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"penulis\",\n      \"properties\": {\n        \"dtype\": \"string\",\n        \"num_unique_values\": 1410,\n        \"samples\": [\n          \"HENDRAYANTO\",\n          \"UMMI MAHSUNAH\",\n          \"Auliya Mufidah\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"judul\",\n      \"properties\": {\n        \"dtype\": \"string\",\n        \"num_unique_values\": 1417,\n        \"samples\": [\n          \"PELAKSANAAN PERNIKAHAN PEREMPUAN HAMIL DILUAR NIKAH DI DESA GRUJUGAN KECAMATAN LARANGAN DAN DESA LARANGAN SLAMPAR KECAMATAN TLANAKAN KABUPATEN PAMEKASAN MENURUT UNDANG-UNDANG REPUBLIK INDONESIA NOMOR\",\n          \"PENANGGULANGAN BALAP LIAR DI KOTA BANGKALAN\",\n          \"TANGGUNG GUGAT PERUSAHAAN ASURANSI YANG MELAKUKAN TINDAKAN WANPRESTASI DALAM ASURANSI\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"abstrak_id\",\n      \"properties\": {\n        \"dtype\": \"string\",\n        \"num_unique_values\": 1408,\n        \"samples\": [\n          \"Dalam organisasi pemerintah, pelayanan kepada masyarakat adalah tujuan utama yang tidak mungkin dapat dihindari karena sudah merupakan kewajiban menyelenggarakan pelayanan dengan menciptakan pelayanan yang terbaik kepada masyarakat. Tujuan pemberian pelayanan publik adalah pemenuhan kebutuhan hak-hak dasar setiap warga negara dan penduduk atas suatu barang, jasa dan atau pelayanan administratif yang disediakan oleh pemerintah yang terkait dengan kepentingan publik. Salah satu jenis pelayanan publik tersebut adalah pelayanan publik di bidang kependudukan dan pencatatan sipil.  \\r\\nTujuan dalam penulisan skripsi ini adalah untuk mengetahui  secara yuridis Akuntabiltas pelayanan publik dalam pembuatan akte kelahiran beserta untuk mengetahui kewenangan Dinas Kependudukan dan Catatan Sipil Kabupaten Bangkalan dalam hal menerbitkan Akta Kelahiran. Sehingga Metode penelitian yang digunakan adalah menggunakan metode penelitian normatif. Adapun pendekatan masalah yang digunakan untuk menjawab rumusan masalah adalah menggunakan pendekatan peraturan perundang-undangan dan pendekatan fakta. \\r\\nSehingga, kesimpulan dari hasil penelitian ini menunjukkan bahwasannya Dinas Kependudukan dan Catatan Sipil Kabupaten Bangkalan yang merupakan Instansi Pelaksana tekhnis dalam asas otonomi daerah dan tugas pembantuan adalah bagian dari penyelenggara pelayanan publik yang melayani urusan kependudukan dan berdasarkan Peraturan  Daerah  Kabupaten Bangkalan  Nomor 6 Tahun 2014 Tentang Perubahan Kedua atas peraturan Daerah Kabupaten Bangkalan Nomor 20 Tahun 2008 Tentang Penyelenggaraan Administrasi Kependudukan berwenang melaksanakan urusan administrasi kependudukan sebagaimana disebutkan dalam pasal 1B huruf c yaitu mencetak, menerbitkan, dan mendistribusikan dokumen kependudukan serta mendokumentasikan hasil pendaftaran penduduk dan pencatatan sipil serta berwenang untuk  memberikan  keabsahan  identitas dan kepastian hukum atas dokumen penduduk untuk setiap peristiwa penting dan peristiwa kependudukan yang dialami oleh penduduk .\\r\\n\\r\\nKata Kunci : Akuntabilitas, Pelayanan Publik, Akte Kelahiran\",\n          \"Hak angket adalah hak untuk melakukan penyelidikan terhadap penyelenggaraan pemerintahan daerah yang berkaitan dengan kebijakan yang dibuat oleh kepala daerah yang memiliki dampak luas dan strategis. Penyelenggaraan pemerintahan daeraeh terdiri dari pemerintah daerah dan DPRD kabupaten/Kota, keberadaan DPRD Kabupaten/Kota dituntut mampu menjadi pengawas atau \\u201ccontrol\\u201d terhadap kebijakan yang dibuat oleh pemerintah daerah, DPRD Kabupaten/kota dan kepala daerah merupakan bagian dari eksekutif, hal tersebut berbeda dengan DPR-Ri yang berperan sebagai lembaga legislatif dan suda sepatutunya melakukan pengawasan terhadap eksekutif, kedudukan DPRD Kabupaten/kota berbeda dengan DPRD-RI yang berada pada tingkatan pusat, sehingga bentuk pengawasan dengan cara angket menimbulkan sebuah masalah yang mengakibatkan urusan rumah tangga daerah tidak akan berjalan secara efektif mengingat kedunya adalah \\u201cmitra kerja\\u201d dalam membangun daerah. Metode penelitian dalam penulisan ini menggunakan penelitian hukum normatif yaitu penelitian hukum yang mencakup penelitian terhadap sinkronisasi peratran perundang-undangan secara vertical dan horizontal, perbandingan hukum dan sejarah hukum. Penelitian ini bersifat deskriptif analisis dengan mengkaji peraturan perundang-undangan. Hasil yang diperoleh dari penelitian tersebut diperlukan reduksi atau meniadakan pemberlakuan hak angket oleh DPRD Kabupaten/Kota, untuk memberikan keseimbangan, stabilitas, dan mencegah terjadinya impeachment terhada kepala daerah oleh DPRD Kabupaten/Kota, sebagaimana dikaetahui bahwasannya keduanya merupakan \\u201cmitra\\u201d dalam menyelenggarakan Pemerintahan Daerah\\nKata kunci: Pemerintahan Daerah, Pemerintah daerah, DPRD Kabupaten/Kota, pemberlakuan, kedudukan, dan hak angket\",\n          \"Dalam skripsi ini yaitu Dikotomi Pemberian Remisi Terhadap Pelaku Tindak Pidana Korupsi Dengan Upaya Pemberantasan Korupsi, untuk melaksanakan hal tersebut diperlukan juga partisipasi atau keikutsertaan masyarakat, baik dengan mengadakan kerjasama dalam pembinaan maupun sikap bersedia menerima kembali narapidana yang telah selesai menjalankan pidananya. Narapidana korupsi mendapatkan hak-haknya didalam Rutan begitupun juga masyarakat wajib mendapatkan haknya, sebagai contoh pemberian remisi yang diberikan terhadap warga binaan korupsi.\\nRemisi adalah pengurangan masa hukuman yang diberikan kepada narapidana dan anak pidana yang telah berkelakuan baik selam menjalani pidana terkecuali yang dipidana mati atau seumur hidup, pemberian remisi kepada warga binaan korupsi memang wajib diberikan karena menyangkut hak mereka sebagai warga binaan tetapi masyarakat juga memerlukan hak mereka yang dikorupsi oleh koruptor, hak mereka lebih berharga.\\nKata kunci : Remisi, Korupsi, Rutan, pemidanaan\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"abstrak_en\",\n      \"properties\": {\n        \"dtype\": \"string\",\n        \"num_unique_values\": 1400,\n        \"samples\": [\n          \"Grant is a voluntarily gift. Grant has a social function to bind hospitality.\\nGrants practice may lead to disputes, such as the grant nullification. As a result, the dispute leads to broke the hospitality binding. One of the legal issues in the grant\\n\\n\\n\\n\\n1 \\n\\n\\n\\nnullification stated in the Verdict of Shariah Court of Gorontalo  Number 9 /Pdt.G\\n/ 2013 / PA.Gtlo. It was the case on conditional grant. Therefore, this study purposed to determine whether the reason of negligence deserved to be the lawsuit argumentation and whether the grant nullification is shariah compliance. This study was categorized as legal reseach and applied  the analytical approach and statute approach. The result of this reseach indicated that the negligence reason of the grant receiver cannot be the legal reason for the lawsuit. Islamic jurisprudence explained that the withdrawal or the nullification of the grant was depicted as like a vomiting dog whose eat its own vomit. The majority opinion of classical Islamic scholars said the nullification of the grant is illegal  while the minority scholars stated that s ruled as avoided is unhave argued the ruling makruh. However, the judge of the verdict approve the argument of the lawsuit and null the grant. The verdict is not shariah compliance because the judges were inconsistent on their verdict consideration, section 210 item (1) Islamic Compilation Law of Indonesia in which rules the maximum grant is 1/3 from the grantor\\u2019s property. In addition, the judges were lack of accuracy in checking the plaintif legal position who has no legal standing.\",\n          \"The legal politics of the formation of the Regional Representative Council is the focal point in this skiripsi, the background of the title election on the legal politics of the establishment of the Regional Representative Council according to the author is very reasonable, because until now the existence of the Regional Representative Council as regional representatives is not visible and tend to fade. Many efforts made by the community to support in order that the Regional Representative Council is still held even voiced the need to amend the fifth of the 1945 Constitution. \\nIt is interesting to examine the legal politics of the establishment of the Regional Representatives Council. Why this Regional Representative Council was formed for what purpose? What is the relationship between the authority of the Regional Representative Council and the House of Representatives? The purpose of this study is to determine what considerations are used and the reasons underlying the formation of the Regional Representative Council. The method used for this research is the historical approach and the formation process undertaken by the People's Consultative Assembly at the post-reproduction session of 1999-2002 and the approach of the legislation. The results of this study say that the purpose of establishing the Regional Representative Council for the acceleration of democratization, safeguarding and strengthening regional ties within the Unitary State of the Republic of Indonesia and improving the accommodation of regional interests.\\n\\nKey terms: (The legal politics of the establishment of the Regional Representative Council, the relationship between the authority of the Regional Representative Council and the House of Representatives).\",\n          \"The right to information is one of the human rights guaranteed by the Constitution In Section 28F of the Constitution of the Republic of Indonesia Year 1945 , Thus , as part of the state is obliged to respect human rights , uphold , and protect and ensure the fulfillment of these rights . Nevertheless the Act No. 14 of 2008 on Public Information there are some exempt information means there is some information that the Act is not allowed to be opened and accessible to the public . The exception is one of the most important aspects of the Act No. 14 of 2008 on Public Information because it defines the limits of the right to information . Restrictions on access to information is a limitation on Human Rights as to which is guaranteed in the Constitution of 1945, because of the exclusion must be based on an objective basis and legally valid and can be accessed and applied proportionately . As a result , the philosophy underlying the exceptions to access to information is very important to be understood . This type of research in this paper is normative research , ie research with writing that is based on an analysis of several legal theories and laws are appropriate and related to the issues in this thesis . The method used to approach the problem in this thesis using two (2 ) approaches , ie . Statute Approach approach is the approach by using legislation and regulation as well as Conseptual Approach Approach is approach to examine the views of legal scholars of the country in which this thesis was made . To ensure that the exclusion clause in the Act - Freedom of Information Act is implemented correctly then there should be a study to determine where the boundaries and categorization of information that must be disclosed to the public or otherwise . because of the exclusion must be based on an objective basis and legally valid and can be accessed and applied proportionately and the philosophy underlying the exceptions to access to information is very important to be understood\\r\\nKeywords : Rights , Public Information , Exceptions\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"pembimbing_pertama\",\n      \"properties\": {\n        \"dtype\": \"string\",\n        \"num_unique_values\": 950,\n        \"samples\": [\n          \"Dr. Wartiningsih., SH., Mhum\",\n          \"Tolib Effendi, S.H.,M.H\",\n          \"Dr. Djulaeka, S.H.,M.Hum.\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"pembimbing_kedua\",\n      \"properties\": {\n        \"dtype\": \"category\",\n        \"num_unique_values\": 203,\n        \"samples\": [\n          \"GATOET POERNOMO, S.H., M.Hum.\",\n          \"Tolib Effendi, SH. MH.\",\n          \"Dr.Wartiningsih,S.H.,M.Hum\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"prodi\",\n      \"properties\": {\n        \"dtype\": \"category\",\n        \"num_unique_values\": 1,\n        \"samples\": [\n          \"Ilmu Hukum\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    }\n  ]\n}","type":"dataframe"}
```
:::
:::

::: {.cell .markdown id="GS42k1S6u7hf"}
**Function All Data 5 pages**
:::

::: {.cell .code id="o7miD7iovSMf"}
``` python
def print_progress(prodi_id, prodi, current_page, total_pages):
    percent = (current_page / total_pages) * 100
    bar_length = 20
    filled_length = int(bar_length * current_page // total_pages)
    bar = '█' * filled_length + '-' * (bar_length - filled_length)
    sys.stdout.write(f'\r[{prodi_id}] {prodi} - Page {current_page}/{total_pages} [{bar}] {percent:.2f}%')
    sys.stdout.flush()
    if current_page == total_pages:
        sys.stdout.write('\n\n')

def pta():
    start_time = time.time()  # mulai hitung waktu

    data = {
        "id": [],
        "penulis": [],
        "judul": [],
        "abstrak id": [],
        "abstrak en": [],
        "pembimbing_pertama": [],
        "pembimbing_kedua": [],
        "prodi": [],
    }

    for i in range(1, 42):  # jumlah prodi
        total_pages = 5  # jumlah page
        prodi_name = None

        for j in range(1, total_pages + 1):  # loop page
            url = f"https://pta.trunojoyo.ac.id/c_search/byprod/{i}/{j}"
            r = requests.get(url)
            soup = BeautifulSoup(r.content, "html.parser")
            jurnals = soup.select('li[data-cat="#luxury"]')

            isii = soup.select_one('div#begin')
            if not isii:
                continue
            prodi_full = isii.select_one('h2').text.strip()
            prodi = prodi_full.replace("Journal Jurusan ", "")
            if not prodi_name:
                prodi_name = prodi

            for jurnal in jurnals:
                link = jurnal.select_one('a.gray.button')['href']

                # ambil ID dari link PTA
                id_match = re.search(r"/detail/(\d+)", link)
                pta_id = id_match.group(1) if id_match else None

                response = requests.get(link)
                soup1 = BeautifulSoup(response.content, "html.parser")
                isi = soup1.select_one('div#content_journal')

                # Judul
                judul = isi.select_one('a.title').text

                # Penulis
                penulis = isi.select_one('span:contains("Penulis")').text.split(' : ')[1]

                # Pembimbing Pertama
                pembimbing_pertama = isi.select_one('span:contains("Dosen Pembimbing I")').text.split(' : ')[1]

                # Pembimbing Kedua
                pembimbing_kedua = isi.select_one('span:contains("Dosen Pembimbing II")').text.split(' :')[1]

                # Abstrak
                paragraf = isi.select('p[align="justify"]')
                abstrak = paragraf[0].get_text(strip=True) if len(paragraf) > 0 else "N/A"
                abstract = paragraf[1].get_text(strip=True) if len(paragraf) > 1 else "N/A"

                # simpan data
                data["id"].append(pta_id)
                data["penulis"].append(penulis)
                data["judul"].append(judul)
                data["pembimbing_pertama"].append(pembimbing_pertama)
                data["pembimbing_kedua"].append(pembimbing_kedua)
                data["abstrak id"].append(abstrak)
                data["abstrak en"].append(abstract)
                data["prodi"].append(prodi)

            # update progress bar
            print_progress(i, prodi_name, j, total_pages)

    df = pd.DataFrame(data)
    df.to_csv("pta.csv", index=False, encoding="utf-8-sig")

    end_time = time.time()
    elapsed = int(end_time - start_time)
    jam, sisa = divmod(elapsed, 3600)
    menit, detik = divmod(sisa, 60)

    # summary
    print("\n✅ Seluruh data berhasil dikumpulkan!")
    print(f"📈 Total entri: {len(df)}")
    print(f"⏱️ Waktu eksekusi: {jam} jam {menit} menit {detik} detik")

    return df
```
:::

::: {.cell .code colab="{\"base_uri\":\"https://localhost:8080/\",\"height\":1000}" id="d3do-N9Cvgik" outputId="24a31e5f-befe-4930-9586-d9378dd038df"}
``` python
pta()
```

::: {.output .stream .stdout}
    [1] Ilmu Hukum - Page 5/5 [████████████████████] 100.00%

    [2] Teknologi Industri Pertanian - Page 5/5 [████████████████████] 100.00%

    [3] Agribisnis - Page 5/5 [████████████████████] 100.00%

    [4] Agroteknologi - Page 5/5 [████████████████████] 100.00%

    [5] Ilmu Kelautan - Page 5/5 [████████████████████] 100.00%

    [6] Ekonomi Pembangunan - Page 5/5 [████████████████████] 100.00%

    [7] Manajemen - Page 5/5 [████████████████████] 100.00%

    [8] Akuntansi - Page 5/5 [████████████████████] 100.00%

    [9] Teknik Industri - Page 5/5 [████████████████████] 100.00%

    [10] Teknik Informatika - Page 5/5 [████████████████████] 100.00%

    [11] Manajemen Informatika - Page 5/5 [████████████████████] 100.00%

    [12] Sosiologi - Page 5/5 [████████████████████] 100.00%

    [13] Ilmu Komunikasi - Page 5/5 [████████████████████] 100.00%

    [14] Psikologi - Page 5/5 [████████████████████] 100.00%

    [15] Sastra Inggris - Page 5/5 [████████████████████] 100.00%

    [16] Ekonomi Syariah - Page 5/5 [████████████████████] 100.00%

    [17] Hukum Bisnis Syariah - Page 5/5 [████████████████████] 100.00%

    [18] Pgsd - Page 5/5 [████████████████████] 100.00%

    [19] Teknik Multimedia Dan Jaringan - Page 5/5 [████████████████████] 100.00%

    [20] Mekatronika - Page 5/5 [████████████████████] 100.00%

    [21] D3 Akuntansi - Page 5/5 [████████████████████] 100.00%

    [22] Magister Manajemen - Page 5/5 [████████████████████] 100.00%

    [23] Teknik Elektro - Page 5/5 [████████████████████] 100.00%

    [24] Magister Ilmu Hukum - Page 5/5 [████████████████████] 100.00%

    [25] Magister Akuntansi - Page 5/5 [████████████████████] 100.00%

    [26] D3 Enterpreneurship - Page 5/5 [████████████████████] 100.00%

    [27] Pendidikan Bhs Dan Sastra Indonesia - Page 5/5 [████████████████████] 100.00%

    [28] Pendidikan Informatika - Page 5/5 [████████████████████] 100.00%

    [29] Pendidikan Ipa - Page 5/5 [████████████████████] 100.00%

    [30] Pgpaud - Page 5/5 [████████████████████] 100.00%

    [31] Sistem Informasi - Page 5/5 [████████████████████] 100.00%

    [32] Teknik Mesin - Page 5/5 [████████████████████] 100.00%

    [33] Teknik Mekatronika - Page 5/5 [████████████████████] 100.00%

    [34] Journal Jurusan - Page 5/5 [████████████████████] 100.00%

    [35] Manajemen Sumberdaya Perairan - Page 5/5 [████████████████████] 100.00%

    [36] Magister Ilmu Ekonomi - Page 5/5 [████████████████████] 100.00%

    [37] Magister Pengelolaan Sumber Daya Alam - Page 5/5 [████████████████████] 100.00%

    [38] Pendidikan Profesi Guru - Page 5/5 [████████████████████] 100.00%

    [39] Magister Pendidikan Dasar - Page 5/5 [████████████████████] 100.00%

    [40] Doktor Pengelolaan Sumber Daya Alam - Page 5/5 [████████████████████] 100.00%

    [41] Doktor Ilmu Manajemen - Page 5/5 [████████████████████] 100.00%


    ✅ Seluruh data berhasil dikumpulkan!
    📈 Total entri: 781
    ⏱️ Waktu eksekusi: 1 jam 25 menit 43 detik
:::

::: {.output .execute_result execution_count="26"}
``` json
{"summary":"{\n  \"name\": \"pta()\",\n  \"rows\": 781,\n  \"fields\": [\n    {\n      \"column\": \"id\",\n      \"properties\": {\n        \"dtype\": \"string\",\n        \"num_unique_values\": 781,\n        \"samples\": [\n          \"140121100020\",\n          \"140121100001\",\n          \"140261100001\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"penulis\",\n      \"properties\": {\n        \"dtype\": \"string\",\n        \"num_unique_values\": 773,\n        \"samples\": [\n          \"Mery Permatasari\",\n          \"Muhammad Trio Maulana Putra\",\n          \"ZAIMUS SILMI\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"judul\",\n      \"properties\": {\n        \"dtype\": \"string\",\n        \"num_unique_values\": 781,\n        \"samples\": [\n          \"KEWENANGAN PENGADILAN AGAMA DALAM MENETAPKAN ISBAT NIKAH TERHADAP PERKAWINAN POLIGAMI SIRI \",\n          \"Kebijakan Kriminalisasi Mengkonsumsi Minuman Beralkohol dalam Rancangan Undang-Undang tentang Larangan Minuman Beralkohol\",\n          \"PENGARUH GREEN MARKETING MIX TERHADAP     KEPUTUSAN PEMBELIAN (Studi Pada Produk Tupperware \\nDi Kabupaten Pamekasan\\n\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"abstrak id\",\n      \"properties\": {\n        \"dtype\": \"string\",\n        \"num_unique_values\": 759,\n        \"samples\": [\n          \"ABSTRAK\\r\\nPenyelesaian sengketa dibedakan menjadi dua diantaranya melalui litigasi dan non litigasi. Jalur litigasi merupakan penyelesaian sengketa melalui Pengadilan. Sedangkan non litigasi merupakan mekanisme penyelesaian sengketa diluar Pengadilan. Penyelesaian sengketa di Pengadilan Agama Bangkalan dilakukan melalui tahapan mediasi sebagai tahap awal dalam anjuran damai dalam menyelesaikan sengketa di Pengadilan Agama Bangkalan. Namun masih banyak sengketa melalui proses mediasi belum berhasil di damaikan. Oleh karena itu proses mediasi di Pengadilan Agama Bangkalan perlu di lakukan penelitian tentang efektivitas mediasi dalam menyelesaikan sengketa di Pengadilan Agama Bangkalan.\\r\\nPenelitian ini bertujuan untuk mengetahui pelaksanan mediasi dalam menyelesaikan sengketa di Pengadilan Agama Bangkalan dan untuk mengetahui efektivitas mediasi dalam menyelesaikan sengketa di Pengadilan Agama Bangkalan. Penulisan skripsi ini menggunakan jenis penelitian lapangan ((field research), dengan menggunakan metode pengumpulan data primer melalui wawancara dengan hakim Pengadilan Agama Bangkalan dan data sekunder melalui kepustakaan. Analisis yang digunakan adalah deskriptif kualitatif. \\r\\nTemuan yang diperoleh dari hasil penelitian ini adalah bahwasannya mediasi dalam menyelesaikan sengketa di Pengadilan Agama Bangkalan masih belum efektif. Hal tersebut dikarenakan beberapa faktor diantaranya tingkat kepatuhan masyarakat dalam menjalani proses mediasi yang sangat rendah dan Para pihak dalam proses mediasi seringkali salah satu dari pihak yang bersengketa atau keduanya merasa paling benar dan mementingkan keegoisan dari  para pihak itu sendiri. Fasilitas dan sarana mediasi di Pengadilan Agama Bangkalan masih kurang memadai, penataan ruangan mediasi kurang nyaman dan fasilitas penunjang didalamnya. Minimnya Hakim dalam memiliki sertifikat mediator dan kurangnya pelatihan mediasi yang dilakukan oleh Mahkamah Agumg RI.\\r\\n\\r\\nkata Kunci: Efektivitas, Mediasi, Sengketa, Pengadilan, Agama\",\n          \"Faishal, SE., 130261100006. THE IMPACT OF SERVICE QUALITY TO CUSTOMER SATISFACTION AND CUSTOMER LOYALTY (Studi Pada PT. POS Indonesia Cabang Pamekasan). ADVISING BY Dr.  H. Muh. Syarif, Drs. Ec., M.Si. and Dr. Mohammad Arief, Drs.Ec., M.Si \\r\\nThis Study wanted to analyne the impact of service quality to customer satisfaction and customer loyalty (study at Indonesia Post Office Branch Pamekasan). This study was used causal method.\\r\\nThe population of this study is the customer of Indonesia Post Office Branch Pamekasan and this study was used sampling design which got 105 respondents. In this study, the writer was used accidental sampling and purposive sampling. The data processing technique was used the Structural Equation Modelling (SEM) to AMOS program version 22.\\r\\nThe service quality consisted of several dimensions. They were: reliability, responsiveness, assurance, tangible, and emphaty. The SEM analysis result had fulfilled criteria of goodness of fit index, it swowed that nilai Chi-square= 85.565; Significance probability = 0,372; RMSEA = 0,017; CMIN/DF = 1.031; TLI = 0,981; CFI = 0,984; GFI = 0,986 dan AGFI = 0,928. The result of analysis shown that service quality (reliability, responsiveness, assurance, emphaty, dan tangibles) is significant influential to customer satisfaction and customer loyalty.\\r\\nKeyword: reliability, responsiveness, assurance, emphaty, tangibles, customer satisfaction, customer loyalty.\",\n          \"Nur Muhammad Ramdhan NRP 09.03.111.00004. Respon Pertumbuhan Dan Produksi Tanaman Kedelai (Glycine max (L) Merr) Kultivar Edamame Terhadap Pengolahan Tanah Dan Takaran Kotoran Ayam Di Tanah Mediteran, dibawah bimbingan Mustika Tripatmasari, SP., M.Si. dan Catur Wasonowati, SP.,M.Si.\\r\\n\\r\\nABSTRAK\\r\\n\\r\\n\\r\\nTanaman kedelai (Glycine max (L) Merr) kultivar edamame adalah tanaman sayuran biji yang dipanen muda. Kedelai edamame menghendaki jenis tanah yang subur yaitu cukup unsur hara dan bahan organik serta sifat fisik tanah yang gembur untuk pertumbuhannya. Tanah mediteran yang merupakan tanah yang terbentuk dari pelapukan batuan kapur dan bersifat tidak subur karena kandungan unsur hara dan bahan organik rendah serta sifat fisik tanah yang tidak gembur. Untuk mendapatkan pertumbuhan dan hasil kedelai edamame yang baik pada tanah mediteran perlu dilakukan pengolahan tanah dan pemberian kotoran ayam. Perlakuan pengolahan tanah dan takaran kotoran ayam ini diharapkan dapat memberikan pertumbuhan dan hasil tanaman edamame yang baik. Penelitian ini bertujuan untuk mengetahui pengaruh pengolahan tanah dan takaran kotoran ayam terhadap pertumbuhan dan hasil tanaman edamame. Penelitian  dilakukan  di Kebun Balai Penelitian Tanaman Hortikultura Socah, Kecamatan Socah, Kabupaten Bangkalan Jawa Timur di Ketinggian \\u00b1 4,2 meter diatas permukaan laut. Penelitian ini berlangsung dari bulan Desember 2012 \\u2013 Maret 2013. Metode  yang  digunakan dengan percobaan Rancangan Acak Kelompok (RAK) faktorial dengan tiga ulangan. Faktor pertama pengolahan tanah terdiri dari 3 taraf (tanpa pengolahan tanah, pengolahan tanah 1 kali dan pengolahan tanah 2 kali), sedangkan faktor kedua terdiri dari 3 taraf (takaran kotoran ayam 4 ton/ha, takaran kotoran ayam 6 ton/ha, dan takaran kotoran ayam 8 ton/ha). Dari hasil penelitian menunjukkan bahwa tidak terjadi interaksi antara perlakuan pengolahan tanah dan takaran kotoran ayam terhadap semua variabel pengamatan. Namun pada masing-masing perlakuan pengolahan tanah berpengaruh nyata terhadap variabel jumlah daun pada umur 42 dan 56 HST, jumlah cabang pada umur 42 HST, dan klorofil daun pada umur 35 HST, sedangkan pada perlakuan takaran kotoran ayam berpengaruh nyata terhadap variabel kandungan klorofil daun pada umur 35, 42, dan 49 HST, jumlah polong, bobot polong, dan polong isi per tanaman.\\r\\n\\r\\n\\r\\nKata kunci: edamame, pengolahan tanah, takaran, kotoran ayam.\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"abstrak en\",\n      \"properties\": {\n        \"dtype\": \"string\",\n        \"num_unique_values\": 745,\n        \"samples\": [\n          \"Mobile technology is an open source game develops very rapidly. Because of the diversity of its variations, mobile game has a lot of interest from various parties. This is the basis for the developers to continue to develop mobile games.\\r\\n In this study, a mobile game to hone thinking ability has been developed to implement the algorithm Prim and Depth First Search. In manufacturing, Prim algorithm is applied at the time of passage of the maze while the Depth First Search algorithm applied at the time of the solution search process.\\r\\n          From the test results it can be concluded that the algorithm Prim and Depth First Search can function properly at screen resolutions of 320 x 480 pixels in the formation of the track and the search for solutions of the maze\\r\\n Key words : Mobile Game, Android, Algoritma Prim, Algoritma Depth First Search.\",\n          \"The role of family dysfunction in trafficking of girls in the Bongas Village, Bongas District, Indramayu Regency that three families studied through a qualitative case study approach to generate a description of the exposure of the professional backgrounds of parents who had become prostitutes turned out to be the basis for the role of family dysfunction that occurs in case, the role of the father as an important subject in the course of trade to the broker child '/ pimp, be it in the bargain price of money as well as cash receipt charmer girls to want to work in accordance with the provisions pimp'. While the girls mother as motivation to want to participate directly in the earnings through the acquisition of work specified pimps, the girls were made as a result of the trading behavior of girls is a family breadwinner who replaced the role of head of the family by prostituting themselves or work as a prostitute who is the impact of the provisions given the pimp their daughters.\\r\\n\\r\\nKeywords: dysfunction, family roles, trafficking of girls\",\n          \"Tabuhan island is an island that has an area of  48,237 m\\u00b2, one of the potential can be developed on the island of Tabuhan are marine ecotourism, among others include: beaches, snorkelling and diving). The purpose of this study was to analyze the island of Tabuhan as marine ecotourism travel category seagrass, diving and snorkeling. The method used to interpret Quickbird imagery and combine data on the suitability for seagrass ecotourism, snorkeling and diving. Value suitability seagrass 49.20 (S1 or area that is less suitable for ecotourism region seagrass). Value suitability snorkling 73.64 (S2 or areas suitable for ecotourism snorkling), value congruence diving 70.37%. Island Tabuhan suitable to be used as diving and snorkeling tourist area but is less suitable for ecotourism seagrass areas.\\r\\n\\r\\n\\r\\nKeywords: ecotourism, Tabuhan Island, land suitability.\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"pembimbing_pertama\",\n      \"properties\": {\n        \"dtype\": \"string\",\n        \"num_unique_values\": 661,\n        \"samples\": [\n          \"Siti Fadjiyana Fitroh, S.Psi., MA\",\n          \"Muhtar Wahyudi. S.Sos. MA \",\n          \"Dr.RM Moch Wispandono,.S.E,.MS\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"pembimbing_kedua\",\n      \"properties\": {\n        \"dtype\": \"string\",\n        \"num_unique_values\": 543,\n        \"samples\": [\n          \"Sri Wahyuni, S.Kom., M.T\",\n          \"Ariesta Kartika Sari S.Si., M.Pd\",\n          \"Fitri Ahmad Kurniawan, SE., M.AK. AK\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"prodi\",\n      \"properties\": {\n        \"dtype\": \"category\",\n        \"num_unique_values\": 34,\n        \"samples\": [\n          \"Ekonomi Syariah\",\n          \"Mekatronika\",\n          \"Pendidikan Informatika\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    }\n  ]\n}","type":"dataframe"}
```
:::
:::

::: {.cell .markdown id="RLSXtmw3Hlq5"}
**Page & Output Link PTA**
:::

::: {.cell .code id="aro6EBZdHTIx"}
``` python
def print_progress(prodi_id, prodi, current_page, total_pages):
    percent = (current_page / total_pages) * 100
    bar_length = 20
    filled_length = int(bar_length * current_page // total_pages)
    bar = '█' * filled_length + '-' * (bar_length - filled_length)
    sys.stdout.write(f'\r[{prodi_id}] {prodi} - Page {current_page}/{total_pages} [{bar}] {percent:.2f}%')
    sys.stdout.flush()
    if current_page == total_pages:
        sys.stdout.write('\n\n')

def pta_links():
    start_time = time.time()  # mulai hitung waktu

    data = {
        "no": [],
        "page": [],
        "link_keluar": []
    }

    no = 1  # nomor urut

    for i in range(1, 42):  # jumlah prodi
        total_pages = 5  # jumlah page
        prodi_name = None

        for j in range(1, total_pages + 1):  # loop page
            url = f"https://pta.trunojoyo.ac.id/c_search/byprod/{i}/{j}"
            r = requests.get(url)
            soup = BeautifulSoup(r.content, "html.parser")
            jurnals = soup.select('li[data-cat="#luxury"]')

            isii = soup.select_one('div#begin')
            if not isii:
                continue
            prodi_full = isii.select_one('h2').text.strip()
            prodi = prodi_full.replace("Journal Jurusan ", "")
            if not prodi_name:
                prodi_name = prodi

            for jurnal in jurnals:
                link = jurnal.select_one('a.gray.button')['href']

                data["no"].append(no)
                data["page"].append(url)          # link page
                data["link_keluar"].append(link)  # link detail
                no += 1

            # update progress bar
            print_progress(i, prodi_name, j, total_pages)

    df = pd.DataFrame(data)
    df.to_csv("pta_links.csv", index=False)

    end_time = time.time()
    elapsed = int(end_time - start_time)
    jam, sisa = divmod(elapsed, 3600)
    menit, detik = divmod(sisa, 60)

    # summary
    print("\n✅ Seluruh link berhasil dikumpulkan!")
    print(f"📊 Total entri: {len(df)}")
    print(f"⏱️ Waktu eksekusi: {jam} jam {menit} menit {detik} detik")

    return df
```
:::

::: {.cell .code colab="{\"base_uri\":\"https://localhost:8080/\",\"height\":1000}" id="SLTvnPxlIIOR" outputId="0a1e2824-6dd7-4c84-f9ea-aca7f5ba4450"}
``` python
pta_links()
```

::: {.output .stream .stdout}
    [1] Ilmu Hukum - Page 5/5 [████████████████████] 100.00%

    [2] Teknologi Industri Pertanian - Page 5/5 [████████████████████] 100.00%

    [3] Agribisnis - Page 5/5 [████████████████████] 100.00%

    [4] Agroteknologi - Page 5/5 [████████████████████] 100.00%

    [5] Ilmu Kelautan - Page 5/5 [████████████████████] 100.00%

    [6] Ekonomi Pembangunan - Page 5/5 [████████████████████] 100.00%

    [7] Manajemen - Page 5/5 [████████████████████] 100.00%

    [8] Akuntansi - Page 5/5 [████████████████████] 100.00%

    [9] Teknik Industri - Page 5/5 [████████████████████] 100.00%

    [10] Teknik Informatika - Page 5/5 [████████████████████] 100.00%

    [11] Manajemen Informatika - Page 5/5 [████████████████████] 100.00%

    [12] Sosiologi - Page 5/5 [████████████████████] 100.00%

    [13] Ilmu Komunikasi - Page 5/5 [████████████████████] 100.00%

    [14] Psikologi - Page 5/5 [████████████████████] 100.00%

    [15] Sastra Inggris - Page 5/5 [████████████████████] 100.00%

    [16] Ekonomi Syariah - Page 5/5 [████████████████████] 100.00%

    [17] Hukum Bisnis Syariah - Page 5/5 [████████████████████] 100.00%

    [18] Pgsd - Page 5/5 [████████████████████] 100.00%

    [19] Teknik Multimedia Dan Jaringan - Page 5/5 [████████████████████] 100.00%

    [20] Mekatronika - Page 5/5 [████████████████████] 100.00%

    [21] D3 Akuntansi - Page 5/5 [████████████████████] 100.00%

    [22] Magister Manajemen - Page 5/5 [████████████████████] 100.00%

    [23] Teknik Elektro - Page 5/5 [████████████████████] 100.00%

    [24] Magister Ilmu Hukum - Page 5/5 [████████████████████] 100.00%

    [25] Magister Akuntansi - Page 5/5 [████████████████████] 100.00%

    [26] D3 Enterpreneurship - Page 5/5 [████████████████████] 100.00%

    [27] Pendidikan Bhs Dan Sastra Indonesia - Page 5/5 [████████████████████] 100.00%

    [28] Pendidikan Informatika - Page 5/5 [████████████████████] 100.00%

    [29] Pendidikan Ipa - Page 5/5 [████████████████████] 100.00%

    [30] Pgpaud - Page 5/5 [████████████████████] 100.00%

    [31] Sistem Informasi - Page 5/5 [████████████████████] 100.00%

    [32] Teknik Mesin - Page 5/5 [████████████████████] 100.00%

    [33] Teknik Mekatronika - Page 5/5 [████████████████████] 100.00%

    [34] Journal Jurusan - Page 5/5 [████████████████████] 100.00%

    [35] Manajemen Sumberdaya Perairan - Page 5/5 [████████████████████] 100.00%

    [36] Magister Ilmu Ekonomi - Page 5/5 [████████████████████] 100.00%

    [37] Magister Pengelolaan Sumber Daya Alam - Page 5/5 [████████████████████] 100.00%

    [38] Pendidikan Profesi Guru - Page 5/5 [████████████████████] 100.00%

    [39] Magister Pendidikan Dasar - Page 5/5 [████████████████████] 100.00%

    [40] Doktor Pengelolaan Sumber Daya Alam - Page 5/5 [████████████████████] 100.00%

    [41] Doktor Ilmu Manajemen - Page 5/5 [████████████████████] 100.00%


    ✅ Seluruh link berhasil dikumpulkan!
    📊 Total entri: 781
    ⏱️ Waktu eksekusi: 0 jam 18 menit 16 detik
:::

::: {.output .execute_result execution_count="28"}
``` json
{"summary":"{\n  \"name\": \"pta_links()\",\n  \"rows\": 781,\n  \"fields\": [\n    {\n      \"column\": \"no\",\n      \"properties\": {\n        \"dtype\": \"number\",\n        \"std\": 225,\n        \"min\": 1,\n        \"max\": 781,\n        \"num_unique_values\": 781,\n        \"samples\": [\n          596,\n          588,\n          544\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"page\",\n      \"properties\": {\n        \"dtype\": \"category\",\n        \"num_unique_values\": 158,\n        \"samples\": [\n          \"https://pta.trunojoyo.ac.id/c_search/byprod/26/4\",\n          \"https://pta.trunojoyo.ac.id/c_search/byprod/10/1\",\n          \"https://pta.trunojoyo.ac.id/c_search/byprod/27/5\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"link_keluar\",\n      \"properties\": {\n        \"dtype\": \"string\",\n        \"num_unique_values\": 781,\n        \"samples\": [\n          \"https://pta.trunojoyo.ac.id/welcome/detail/140121100020\",\n          \"https://pta.trunojoyo.ac.id/welcome/detail/140121100001\",\n          \"https://pta.trunojoyo.ac.id/welcome/detail/140261100001\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    }\n  ]\n}","type":"dataframe"}
```
:::
:::

::: {.cell .markdown id="uWyyGvQXekHw"}
**2. Crawling Berita www.cnnindonesia.com**
:::

::: {.cell .markdown id="tNpnrpjHgv4w"}
**Library**
:::

::: {.cell .code id="GCM72AYNez2r"}
``` python
import requests
from bs4 import BeautifulSoup
import pandas as pd
import time, re, sys
```
:::

::: {.cell .markdown id="0r_wF0IAg1GB"}
**Function**
:::

::: {.cell .code id="3JA0s4o-fOOQ"}
``` python
# Progress bar manual
def print_progress(current_page, total_pages):
    percent = (current_page / total_pages) * 100
    bar_length = 20
    filled_length = int(bar_length * current_page // total_pages)
    bar = '█' * filled_length + '-' * (bar_length - filled_length)
    sys.stdout.write(f'\rPage {current_page}/{total_pages} [{bar}] {percent:.2f}%')
    sys.stdout.flush()
    if current_page == total_pages:
        sys.stdout.write('\n\n')
```
:::

::: {.cell .code id="fLSBpfe8fomq"}
``` python
# Ambil isi berita CNN
def get_article_content(url):
    r = requests.get(url, headers={"User-Agent": "Mozilla/5.0"})
    soup = BeautifulSoup(r.text, "html.parser")

    paragraphs = []
    content_div = soup.find("div", class_="detail-text")
    if content_div:
        for p in content_div.find_all("p"):
            text = p.get_text(strip=True)
            if text and not text.lower().startswith("baca juga"):
                paragraphs.append(text)
    return " ".join(paragraphs)
```
:::

::: {.cell .code colab="{\"base_uri\":\"https://localhost:8080/\"}" id="uCPZb7iHfrxr" outputId="4e6c0089-e927-4b59-c4b4-bba9d1c3a786"}
``` python
# Scraping CNN Indonesia
def berita_cnn(pages=1):
    start_time = time.time()

    BASE_URL = "https://www.cnnindonesia.com/indeks?page={}"

    data = {
        "id_berita": [],
        "judul_berita": [],
        "isi_berita": [],
        "kategori_berita": []
    }

    counter = 1  # mulai id dari 1

    for page in range(1, pages+1):
        url = BASE_URL.format(page)
        r = requests.get(url, headers={"User-Agent": "Mozilla/5.0"})
        soup = BeautifulSoup(r.text, "html.parser")

        articles = soup.select("article a")
        for a in articles:
            link = a.get("href")
            title = a.get_text(strip=True)

            if not link or not title:
                continue
            if not link.startswith("https://www.cnnindonesia.com/"):
                continue

            # Ambil kategori dari URL
            try:
                kategori = link.split("/")[3]
            except:
                kategori = "unknown"

            try:
                content = get_article_content(link)
            except:
                content = ""

            data["id_berita"].append(counter)
            data["judul_berita"].append(title)
            data["isi_berita"].append(content)
            data["kategori_berita"].append(kategori)

            counter += 1  # naikkan id

        print_progress(page, pages)

    df = pd.DataFrame(data)
    df.to_csv("cnnindonesia_berita4.csv", index=False, encoding="utf-8-sig")

    end_time = time.time()
    elapsed = int(end_time - start_time)
    jam, sisa = divmod(elapsed, 3600)
    menit, detik = divmod(sisa, 60)

    print("\n✅ Seluruh data berhasil dikumpulkan!")
    print(f"📈 Total entri: {len(df)}")
    print(f"⏱️ Waktu eksekusi: {jam} jam {menit} menit {detik} detik")

    return df

# Contoh pemanggilan
df_cnn = berita_cnn(pages=100)
```

::: {.output .stream .stdout}
    Page 100/100 [████████████████████] 100.00%


    ✅ Seluruh data berhasil dikumpulkan!
    📈 Total entri: 1000
    ⏱️ Waktu eksekusi: 0 jam 10 menit 32 detik
:::
:::

::: {.cell .code colab="{\"base_uri\":\"https://localhost:8080/\"}" id="gbScvf0Hk9wd" outputId="4ba3ff4f-04da-4f13-afc9-b9d4ed3599b6"}
``` python
from google.colab import drive
drive.mount('/content/drive')
```

::: {.output .stream .stdout}
    Mounted at /content/drive
:::
:::

::: {.cell .code id="L0ER6uAbmZAK"}
``` python
import requests
from bs4 import BeautifulSoup
import pandas as pd
import time, re, sys
```
:::

::: {.cell .code colab="{\"base_uri\":\"https://localhost:8080/\",\"height\":646}" id="zZvv1Zr2f3rN" outputId="47a81175-76ea-4bd1-9269-8cff71f2bdbf"}
``` python
berita_path ="/content/drive/MyDrive/PPW/output/cnnindonesia_berita4.csv"
berita = pd.read_csv(berita_path, index_col="id_berita")

berita
```

::: {.output .execute_result execution_count="4"}
``` json
{"summary":"{\n  \"name\": \"berita\",\n  \"rows\": 1000,\n  \"fields\": [\n    {\n      \"column\": \"id_berita\",\n      \"properties\": {\n        \"dtype\": \"number\",\n        \"std\": 288,\n        \"min\": 1,\n        \"max\": 1000,\n        \"num_unique_values\": 1000,\n        \"samples\": [\n          522,\n          738,\n          741\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"judul_berita\",\n      \"properties\": {\n        \"dtype\": \"string\",\n        \"num_unique_values\": 982,\n        \"samples\": [\n          \"Airlangga Buka Suara soal PHK di Gudang Garam: Karena ModernisasiEkonomi\\u2022 2 hari yang lalu\",\n          \"Puncak Bogor Raih Peringkat Ke-3 Destinasi Pedesaan Terbaik AsiaGaya Hidup\\u2022 2 hari yang lalu\",\n          \"Pajak Kendaraan di Indonesia Salah Satu Tertinggi di DuniaOtomotif\\u2022 2 hari yang lalu\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"isi_berita\",\n      \"properties\": {\n        \"dtype\": \"string\",\n        \"num_unique_values\": 938,\n        \"samples\": [\n          \"Polres Metro Jakarta Timur kembali menjadwalkan ulang klarifikasi terhadap musisiSherina Munafuntuk dimintai terkait unggahannya di media sosial soal penyelamatan seekor kucing milik anggota DPR RIUya Kuya, Jumat (12/9). Sherina mulanya dijadwalkan untuk diklarifikasi pada Senin (8/9), namun ia berhalangan hadir. Sherina kembali dijadwalkan diklarifikasi pada Selasa (9/9), namun masih belum bisa memenuhi panggilan. \\\"(Pemanggilan ulang) Jumat besok jam 10.00 WIB pagi,\\\" kata Kapolres Metro Jakarta Timur, Kombes Alfian Nurrizal kepada wartawan, Rabu (10/9). ADVERTISEMENT SCROLL TO CONTINUE WITH CONTENT Disampaikan Alfian, penjadwalkan ulang klarifikasi ini berdasarkan permintaan Sherina melalui kuasa hukumnya. Namun, Alfian tak membeberkan soal alasan Sherina meminta penjadwalan ulang. \\\"Yang meminta jadwal ulang lawyernya itu. Katanya penyidik saya, ini ke penyidik, jadi kemarin yang bersangkutan enggak bisa hadir, minta dijadwalkan lagi, sehingga kita jadwalkan hari Jumat, kita layangkan surat lagi Jumat gitu,\\\" tutur dia. Sebelumnya, Sherina Munaf membagikan kabar terbaru soal penyelamatan kucing milik Uya Kuya bernama Lili yang sudah ditemukan. \\\"Salah satu kucing dari rumah Uya Kuya ada yang rescue dan semalaman saya dan @indiradiandra sudah koordinasi langsung dengan rescuer. Pagi ini dijemput dan sekarang kucing posisi aman, sedang sayafoster. Ini hanya satu ekor dari kemungkinan 16-20an ekor kucing yangdibreedingdi lokasi tersebut,\\\" tulis Sherina. Sherina juga mendeskripsikan kondisi kucing yang diduga milik Uya Kuya tersebut. \\\"Kondisi: sangat kurus, tulang-tulangnya berasa banget kalau lagi di pet badannya. Untuk para pet owners,pleasesebisa mungkin ADOPTdon'tSHOP, steril kucingnya, kalau tak mampu rawat tak usah pelihara,\\\" lanjut unggahan Sherina. Dalam kasus penjarahan rumah Uya Kuya, Polres Metro Jakarta Timur diketahui telah menetapkan 12 orang sebagai tersangka. Para tersangka ini memiliki peran berbeda, mulai dari provokator, pelaku penjarahan dan penyerangan kepada petugas.\",\n          \"Bagi penderitakolesteroldan asam urat, menjaga polamakansangatlah penting. Makanan yang tinggi kolesterol dan purin harus dihindari karena dapat memperburuk kondisi kesehatan. Namun, hampir semua makanan hewani mengandung kolesterol dan purin dalam kadar tertentu, sehingga Anda harus lebih jeli dalam memilih makanan ini. Salah satu sumber protein yang banyak digemari adalah ikan. Namun tidak semua jenis ikan cocok untuk dikonsumsi oleh penderita kolesterol dan asam urat. ADVERTISEMENT SCROLL TO CONTINUE WITH CONTENT Untungnya, ada beberapa ikan yang boleh dimakan penderita kolesterol dan asam urat karena kandungan kolesterol dan purinnya relatif rendah. Meski demikian, Anda tetap harus bijak dalam mengatur porsi makan agar manfaat nutrisi dari ikan tetap optimal tanpa membahayakan kesehatan. MenurutCleveland Clinic, makanan yang dianggap rendah purin, adalah makanan yang mengandung kurang dari 100 miligram (mg) purin per 100 gram. Lantas, bagaimana dengan batas kolesterol yang aman? MengutipUCSF Health, bagi orang dengan faktor risiko penyakit jantung, sebaiknya tidak mengonsumsi lebih dari 200 mg kolesterol per hari. Jika Anda tidak ada faktor risiko penyakit jantung, sebaiknya tetap batasi konsumsi makanan berkolesterol, yakni tidak lebih dari 300 mg per hari. Merangkum dari berbagai sumber, berikut ini daftar lima ikan yang tergolong aman dan boleh dikonsumsi oleh penderita kolesterol dan asam urat: Ikan kod merupakan salah satu pilihan terbaik untuk penderita kolesterol dan asam urat. MengutipFoodstruct, kod mengandung sekitar 66 mg kolesterol dan 71 mg purin per porsi 100 gram. Selain itu, ikan kod kaya akan asam lemak omega-3 yang bermanfaat menurunkan kadar trigliserida dan meningkatkan kolesterol baik (HDL). Omega-3 juga membantu mencegah penggumpalan trombosit yang dapat menyumbat pembuluh darah dan menjaga kesehatan jantung tetap optimal. Ikan haddock yang masih satu famili dengan kod, memiliki rasa ringan dan tekstur daging yang lembut serta lembap. Ikan ini mengandung 55 mg kolesterol dan 59 mg purin per 100 gram, menjadikannya pilihan yang aman untuk penderita kolesterol serta asam urat. Selain itu,haddock kaya akan vitamin B12, B6, dan B3 yang sangat penting untuk metabolisme energi dan fungsi saraf. Ikan perch atau kerakap mengandung 115 mg kolesterol dan 67 mg purin per 100 gram. Meski tampak banyak, jumlah kolesterol ini 3,2 kali lebih rendah dibandingkan telur, sehingga aman dikonsumsi dalam batas wajar.\\u00a0Perch juga tinggi akan asam lemak omega-3 dan omega-6 yang bermanfaat untuk kesehatan jantung, mata, serta sistem imun tubuh. Tilapia merupakan ikan air tawar yang populer dan mengandung 57 mg kolesterol serta 60 mg purin per 100 gram. Selain rendah kalori, tilapia juga kaya protein dengan kandungan sekitar 26 gram protein dalam setiap 100 gram. MelansirHealthline, ikan ini juga menyediakan berbagai vitamin dan mineral penting seperti niasin, vitamin B12, fosfor, selenium, dan kalium yang mendukung fungsi tubuh secara menyeluruh. Pike, atau yang dikenal sebagai ikan tombak, mengandung sekitar 39 mg kolesterol dan 59 mg purin per 100 gram. Ini menjadikannya ikan dengan kadar kolesterol dan purin yang sangat rendah. Ikan ini juga rendah lemak jenuh, sehingga baik untuk kesehatan kardiovaskular. Selain itu,pike merupakan sumber protein berkualitas tinggi, kaya akan niasin, serta mineral seperti kalium dan fosfor. Berbagai vitamin dan mineral tersebut dapat membantu mengatur detak jantung, tekanan darah, kesehatan tulang, hingga fungsi saraf. Memilih ikan yang boleh dimakan penderita kolesterol dan asam urat sangat penting untuk mencegah komplikasi penyakit. Daftar ikan di atas merupakan pilihan yang tergolong aman karena kandungan kolesterol dan purinnya rendah. Meski demikian, konsumsi tetap harus dilakukan secara bijak dengan memperhatikan porsi agar manfaat kesehatan tetap optimal.\",\n          \"Timnas Indonesiaakan menghadapiLebanonpada FIFA Matchday September 2025. Berikut prediksi Indonesia vs Lebanon versi redaksi CNNIndonesia. Duel Indonesia vs Lebanon akan digelar di Stadion Gelora Bung Tomo, Jawa Timur, Senin (8/9) malam WIB. Pertandingan ini akan disiarkan secara langsung di televisi pukul 20.30 WIB. Ini menjadi laga uji coba kedua Indonesia di FIFA Matchday September 2025. Sebelumnya skuad Garuda berhasil meraih kemenangan 6-0 atas Taiwan. ADVERTISEMENT SCROLL TO CONTINUE WITH CONTENT Pelatih Patrick Kluivert berpeluang menurunkan skuad terbaiknya saat melawan Lebanon. Beberapa pemain pilar yang disimpan seperti kapten Jay Idzes dan Justin Hubner siap dimainkan. Duet Thom Haye dan Joey Pelupessy juga kemungkinan bakal diandalkan sejak menit awal untuk menghidupkan lini tengah. Begitu pula dengan Kevin Diks dan Calvin Verdonk. Kedua bek sayap ini sengaja diistirahatkan saat Indonesia vs Taiwan. Duel kontra Lebanon menjadi penting bagi Kluivert. Selain bisa mendongkrak ranking FIFA Indonesia, uji coba ini menjadi persiapan terakhir jelang tampil di babak keempat Kualifikasi Piala Dunia 2026 melawan Irak dan Arab Saudi. Racikan strategi Patrick Kluivert menarik dinanti. Apakah tim Merah Putih mampu tampil impresif melawan Lebanon atau malah terpeleset? Berikut prediksi Indonesia vs Lebanon yang dirangkum redaksi CNNIndonesia.com: Timnas Indonesia telah menunjukkan permainan apik saat mengalahkan Taiwan 6-0 pada pertandingan uji coba terakhir. Kini para pemain Indonesia dalam kepercayaan diri yang baik dan pelatih yang dikepalai Patrick Kluivert juga sepertinya sudah menemukan cara terbaik timnya untuk menghadapi Lebanon. Kehadiran Mauro Zijlstra dan Miliano Jonathans mampu memberikan warna baru pada permainan Timnas Indonesia. Khususnya Miliano Jonathans yang disebut-sebut sebagai Arjen Robben-nya Indonesia yang mampu tampil memukau saat lawan Taiwan. Saya memprediksi Timnas Indonesia akan tampil onfire dan mengalahkan Lebanon 3-0. Baca di halaman berikutnya>>> Lupakan kemenangan atas Taiwan, laga Indonesia vs Lebanon merupakan uji coba sesungguhnya. Pemain seperti Jay Idzes, Kevin Diks, Calvin Verdonk, dan Ragnar Oratmangoen yang hanya duduk di tribune penonton saat melawan Taiwan, dipastikan akan bermain dan menjadi starter. Patrick Kluivert harus mengeluarkan kekuatan terbaik, sudah tidak ada lagi eksperimen. Meski begitu, Miliano Jonathans layak menjadi starter. Lebanon mengandalkan fisik dan kecepatan. Itu sebabnya Indonesia harus waspada dengan permainan transisi yang cepat. Selain itu Indonesia harus cepat dalam melakukan pressing saat kehilangan bola. Meski Lebanon punya ranking FIFA di atas Indonesia, namun saya prediksi Tim Garuda punya peluang bagus untuk menang di Surabaya. Skor akhir 2-1 untuk Indonesia. Para pemain yang selama ini nyaman menghuni posisi di tim inti mungkin agak sedikit panas dengan penampilan para pemain pelapis di laga lawan Taiwan. Karena itu mereka akan gantian unjuk gigi di laga lawan Lebanon. Lebanon bisa dibilang punya kekuatan yang selevel dengan Indonesia dan bakal jadi alat ukur yang pas sebelum duel lawan Irak dan Arab Saudi. Indonesia bakal tampil enerjik di laga nanti dan meraih kemenangan 2-0 di akhir laga. Lebanon jadi satu-satunya lawan sepadan bagi Timnas Indonesia dalam FIFa Matchday kali ini. Karena itu, laga ini tidak akan mudah bagi Skuad Garuda. Dengan memainkan pemain utama, Indonesia sepertinya bakal memiliki sejumlah kesempatan dalam menguasai bola dan permainan. Akan tetapi, perlawanan juga diberikan Lebanon. Meski begitu, persoalan Indonesia sejak laga-laga sebelumnya tetap sama, yakni finishing. Dengan kehadiran Mauro Zijlstra dan Miliano Jonathan, Indonesia akan punya peluang gol. Kalau tidak bisa memanfaatkan gol, Indonesia bakal kesulitan lawan Lebanon. Sebaliknya, dengan finishing yang baik, Indonesia bisa menang 2-1. [Gambas:Photo CNN] Lebanon memang punya kualitas lebih baik dari Taiwan. Namun, para pemain Garuda tengahon firedan siap unjuk gigi demi mendapat tempat bermain di babak keempat Kualifikasi Piala Dunia 2026. Thom Haye akan kembali jadi pengatur ritme permainan di lini tengah. Gelandang Persib itu bakal berupaya memanjakan pemain baru, Miliano Jonathans dan Mauro Zijlstra yang berpeluang diberi durasi bermain lebih banyak. Dengan motivasi berlipat ganda, saya prediksi Indonesia mampu raih kemenangan 3-1 atas Lebanon di Stadion GBT. Jika Taiwan ibarat hanya pemanasan, maka Lebanon jadi lawan pas buat menguji kemampuan melawan Irak dan Arab Saudi pada Kualifikasi Piala Dunia 2026. Dibanding Taiwan, Lebanon tampaknya bakal lebih menyulitkan. Lantaran hal tersebut, Kluivert kemungkinan bakal menurunkan pemain-pemain yang akan menjadi starter pada bulan depan dalam laga melawan Lebanon. Saya prediksi Indonesia bisa menang dengan perjuangan lebih. Skor 3-1. [Gambas:Video CNN]\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"kategori_berita\",\n      \"properties\": {\n        \"dtype\": \"category\",\n        \"num_unique_values\": 10,\n        \"samples\": [\n          \"edukasi\",\n          \"internasional\",\n          \"ekonomi\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    }\n  ]\n}","type":"dataframe","variable_name":"berita"}
```
:::
:::

::: {.cell .markdown id="URr6jXKlhEDk"}
**Page & Link Output Berita**
:::

::: {.cell .code colab="{\"base_uri\":\"https://localhost:8080/\"}" id="d56_3SBkf60u" outputId="27b49e35-c156-43f9-b80b-c1abcf521295"}
``` python
import requests
from bs4 import BeautifulSoup
import pandas as pd
import time, sys

# progress bar
def print_progress(current_page, total_pages):
    percent = (current_page / total_pages) * 100
    bar_length = 20
    filled_length = int(bar_length * current_page // total_pages)
    bar = '█' * filled_length + '-' * (bar_length - filled_length)
    sys.stdout.write(f'\rPage {current_page}/{total_pages} [{bar}] {percent:.2f}%')
    sys.stdout.flush()
    if current_page == total_pages:
        sys.stdout.write('\n\n')

# fungsi untuk kumpulkan link berita CNN Indonesia
def berita_links(pages=1):
    start_time = time.time()

    BASE_URL = "https://www.cnnindonesia.com/indeks?page={}"

    data = {
        "id_berita": [],
        "page": [],
        "link_keluar": []
    }

    counter = 1
    for page in range(1, pages+1):
        url = BASE_URL.format(page)
        r = requests.get(url, headers={"User-Agent": "Mozilla/5.0"})
        soup = BeautifulSoup(r.text, "html.parser")

        articles = soup.select("article a")
        for a in articles:
            link = a.get("href")

            if not link or not link.startswith("https://www.cnnindonesia.com/"):
                continue

            data["id_berita"].append(counter)
            data["page"].append(url)       # halaman indeks
            data["link_keluar"].append(link)  # link detail berita

            counter += 1

        print_progress(page, pages)

    df = pd.DataFrame(data)
    df.to_csv("cnnindonesia_links1.csv", index=False, encoding="utf-8-sig")

    end_time = time.time()
    elapsed = int(end_time - start_time)
    jam, sisa = divmod(elapsed, 3600)
    menit, detik = divmod(sisa, 60)

    print("\n✅ Seluruh link berhasil dikumpulkan!")
    print(f"📈 Total entri: {len(df)}")
    print(f"⏱️ Waktu eksekusi: {jam} jam {menit} menit {detik} detik")

    return df

# contoh pemanggilan: ambil 100 halaman pertama
df_links = berita_links(pages=100)
```

::: {.output .stream .stdout}
    Page 100/100 [████████████████████] 100.00%


    ✅ Seluruh link berhasil dikumpulkan!
    📈 Total entri: 1000
    ⏱️ Waktu eksekusi: 0 jam 1 menit 0 detik
:::
:::

::: {.cell .code colab="{\"base_uri\":\"https://localhost:8080/\",\"height\":649}" id="2_YssHGhgIz0" outputId="7483d068-3208-467f-b96e-6852ad70dd0a"}
``` python
link_path = "/content/drive/MyDrive/PPW/output/cnnindonesia_links1.csv"
link = pd.read_csv(link_path, index_col="id_berita")

link
```

::: {.output .execute_result execution_count="6"}
``` json
{"summary":"{\n  \"name\": \"link\",\n  \"rows\": 1000,\n  \"fields\": [\n    {\n      \"column\": \"id_berita\",\n      \"properties\": {\n        \"dtype\": \"number\",\n        \"std\": 288,\n        \"min\": 1,\n        \"max\": 1000,\n        \"num_unique_values\": 1000,\n        \"samples\": [\n          522,\n          738,\n          741\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"page\",\n      \"properties\": {\n        \"dtype\": \"category\",\n        \"num_unique_values\": 100,\n        \"samples\": [\n          \"https://www.cnnindonesia.com/indeks?page=84\",\n          \"https://www.cnnindonesia.com/indeks?page=54\",\n          \"https://www.cnnindonesia.com/indeks?page=71\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    },\n    {\n      \"column\": \"link_keluar\",\n      \"properties\": {\n        \"dtype\": \"string\",\n        \"num_unique_values\": 976,\n        \"samples\": [\n          \"https://www.cnnindonesia.com/ekonomi/20250910111956-625-1272054/pertamina-resmi-groundbreaking-pilot-plant-green-hydrogen-di-ulubelu\",\n          \"https://www.cnnindonesia.com/teknologi/20250909102826-199-1271545/benarkah-ketindihan-saat-tidur-ulah-setan-ini-penjelasan-ilmiahnya\",\n          \"https://www.cnnindonesia.com/olahraga/20250910123713-170-1272098/peminat-tetap-tinggi-audisi-pb-djarum-diikuti-1729-peserta\"\n        ],\n        \"semantic_type\": \"\",\n        \"description\": \"\"\n      }\n    }\n  ]\n}","type":"dataframe","variable_name":"link"}
```
:::
:::
