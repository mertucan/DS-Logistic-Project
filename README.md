# Lojistik Veri Analizi ve Tahminleme Projesi - Teknik Dokümantasyon

Bu doküman, lojistik operasyonlarına ait büyük ölçekli bir veri seti üzerinde gerçekleştirilen uçtan uca veri bilimi projesinin detaylı anlatımını, kullanılan yöntemlerin teknik gerekçelerini ve elde edilen sonuçların yorumlanmasını içerir.

---

## 1. Proje Hakkında ve Veri Seti

### 1.1. Projenin Amacı
Bu projenin temel amacı, ham lojistik verilerini işleyerek anlamlı bilgiler çıkarmak, teslimat sürelerini tahmin eden makine öğrenmesi modelleri geliştirmek ve kargo profillerini segmentlere ayırarak operasyonel verimliliği artıracak stratejiler belirlemektir.

### 1.2. Veri Seti Hikayesi
Kullanılan veri seti (**logistics_shipments.csv**), 750.000 satır ve 7 sütundan oluşan sentetik veya anonimleştirilmiş gerçek hayat lojistik verilerini içerir.

**Değişkenler:**
*   **shipment_id:** Kargonun benzersiz kimlik numarası (Analiz dışı bırakıldı).
*   **origin / destination:** Kargonun çıkış ve varış noktaları.
*   **weight_kg:** Kargonun ağırlığı.
*   **volume_m3:** Kargonun hacmi.
*   **transport_mode:** Taşıma şekli (AIR, SEA, ROAD).
*   **delivery_days:** Teslimatın kaç gün sürdüğü (**Hedef Değişken**).

---

## 2. Kullanılan Yöntemler ve Teknik Gerekçeler

Projede uygulanan her teknik, belirli bir veri bilimi problemini çözmek için seçilmiştir. Aşağıda bu seçimlerin nedenleri detaylandırılmıştır:

### 2.1. Neden Accuracy, F1-Score Kullanılmadı?
Proje isterlerinde bu metrikler geçmesine rağmen, bu projede **kullanılmamıştır**.
*   **Sebep:** Bizim problemimiz bir **Regresyon (Tahminleme)** problemidir. Yani "Teslimat 3.4 gün sürer" gibi sayısal bir değer tahmin etmeye çalışıyoruz.
*   **Açıklama:** *Accuracy (Doğruluk)* ve *F1-Score*, bir verinin sınıfını (Kedi mi? Köpek mi? Hasta mı? Değil mi?) tahmin eden **Sınıflandırma (Classification)** problemleri için kullanılır. Regresyon problemlerinde başarının ölçütü, tahminin gerçeğe ne kadar yakın olduğudur (Hata miktarı). Bu yüzden **MSE (Mean Squared Error)** ve **R2 Score** kullanılmıştır.

### 2.2. IQR (Interquartile Range) ile Outlier Baskılama
*   **Neden:** Lojistik verilerinde bazen sistemsel hatalardan dolayı aşırı yüksek ağırlık veya hacim değerleri olabilir. Bu uç değerler ortalamayı saptırır ve modelin yanlış öğrenmesine sebep olur.
*   **Uygulama:** Veriyi silmek yerine, alt ve üst sınırların dışındaki değerler o sınırlara eşitlenmiştir (Capping). Böylece veri kaybı yaşanmadan gürültü temizlenmiştir.

### 2.3. One-Hot Encoding
*   **Neden:** Bilgisayarlar ve matematiksel modeller kelimelerden (AIR, ROAD) anlamaz, sayılarla çalışır.
*   **Uygulama:** `transport_mode` sütunu `mode_AIR`, `mode_ROAD` gibi sütunlara ayrılarak 1 (var) ve 0 (yok) değerlerine dönüştürülmüştür.

---

## 3. Proje Adımları ve Kod Açıklamaları

### Bölüm 1: Veri Hazırlama (Data Preparation)

Bu aşamada ham veri işlenerek analize hazır hale getirilmiştir.

1.  **Eksik Değer Analizi:** Veri setinde boş değer olmamasına rağmen, kod gelecekte oluşabilecek boşluklara karşı dayanıklı hale getirilmiştir. Sayısal veriler ortalama, kategorik veriler mod ile doldurulacak şekilde kurgulanmıştır.
2.  **Feature Engineering (Öznitelik Mühendisliği):**
    *   **Density (Yoğunluk):** `Ağırlık / Hacim` formülüyle yeni bir sütun üretilmiştir. Bu, lojistikte "desi" kavramına benzer ve yükün niteliğini belirler.
    *   **Not:** Hacim 0 olduğunda ortaya çıkan sonsuz (inf) değerler temizlenmiş ve ortalama ile doldurulmuştur.
3.  **Ölçeklendirme (MinMax Scaling):** Ağırlık (kg) ile Hacim (m3) birimleri farklıdır. Biri 5000 olabilirken diğeri 0.5 olabilir. Modellerin büyük sayıya odaklanmasını engellemek için tüm veriler 0-1 arasına sıkıştırılmıştır.

### Bölüm 2: Keşifçi Veri Analizi (EDA)

Veriyi görselleştirerek anlamaya çalıştığımız bölümdür.

*   **Histogramlar:** Verinin dağılımını (Normal mi? Çarpık mı?) gösterir. Teslimat sürelerinin 2-10 gün arasında toplandığı görülmüştür.
*   **Korelasyon (Heatmap):** Değişkenler arasındaki ilişkiyi gösterir. Örneğin Ağırlık arttıkça Teslimat Süresi artıyor mu? (Analizde çok güçlü bir doğrusal ilişki çıkmamıştır, bu da problemin zorluğunu gösterir).
*   **Boxplot:** Aykırı değerleri (Outliers) görsel olarak tespit etmemizi sağlamıştır.

### Bölüm 3: Modelleme (Algoritmalar)

#### A. K-Means Clustering (Gözetimsiz Öğrenme)
*   **Amaç:** Verideki desenleri bulmak. Etiket (Y) yoktur.
*   **Sonuç:** Kargolar **Ağırlık** ve **Hacimlerine** göre 4 farklı kümeye ayrılmıştır (Örn: Hafif-Hacimli, Ağır-Kompakt vb.). Bu, operasyonel planlama (hangi araca hangi yük konulacak) için kullanılır.

#### B. Linear Regression (Gözetimli Öğrenme - Baseline)
*   **Amaç:** En basit modelle bir başlangıç noktası belirlemek.
*   **Sonuç:** Modelin R2 skoru 0 civarında çıkmıştır. Bu, değişkenler arasında "Doğrusal" (Linear) bir ilişki olmadığını, yani ağırlık artınca sürenin direkt artmadığını gösterir. Daha karmaşık bir modele ihtiyaç vardır.

#### C. Decision Tree Regressor (Gözetimli Öğrenme - Non-Linear)
*   **Amaç:** Doğrusal olmayan karmaşık ilişkileri modellemek. Karar ağaçları "Eğer ağırlık > 10 ve rota = AIR ise süre = 2 gün" gibi kurallar zinciri oluşturur.
*   **Optimization (GridSearchCV):** Modelin aşırı öğrenmesini (Overfitting) engellemek için ağacın maksimum derinliği (`max_depth`) ve dallanma kriterleri optimize edilmiştir.

---

## 4. Çıktıların Yorumlanması

### Model Performans Tablosu
| Model | MSE (Hata) | R2 Score | Yorum |
|-------|------------|----------|-------|
| Linear Regression | Yüksek | ~0 | Veri seti doğrusal modellemeye uygun değil. |
| Decision Tree | Orta | Düşük | Veri setindeki gürültü nedeniyle tahmin zorluğu yaşanıyor ancak en iyi sonucu optimize edilmiş hali veriyor. |

*   **MSE (Ortalama Karesel Hata):** Tahminlerimizin gerçek değerden ortalama ne kadar saptığının karesidir. Düşük olması iyidir.
*   **R2 Score:** Modelin verideki değişimi ne kadar açıkladığıdır. 1'e yakın olması iyidir.

### Grafikler
1.  **Residual Plot (Hata Dağılımı):** Noktaların 0 çizgisi etrafında rastgele dağılması, modelin sistematik bir hata yapmadığını, hatanın rastgele olduğunu gösterir. Bu istatistiksel olarak istenen bir durumdur.
2.  **Cluster Scatter Plot:** Renkli noktalarla ayrılmış grafik, K-Means algoritmasının kargoları fiziksel özelliklerine göre ne kadar net ayırt ettiğini gösterir. Kümelerin belirgin şekilde ayrılması modelin başarısını kanıtlar.

---

## 5. Sonuç

Bu proje, ham veriden başlayıp model optimizasyonuna kadar giden tam bir veri bilimi döngüsünü (Pipeline) içermektedir.

*   Veri temizlenmiş ve **Regresyon** problemine uygun hale getirilmiştir.
*   Yanlış metrikler (Accuracy) yerine doğru metrikler (MSE, R2) kullanılmıştır.
*   Lojistik operasyonları için hem **Tahminleme** (Kaç günde gider?) hem de **Segmentasyon** (Hangi tür yük?) çözümleri üretilmiştir.

