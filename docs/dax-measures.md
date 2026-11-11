# DAX Ölçüleri ve Power Query Dönüşüm Referansı

## 1. Power Query (M) Veri Temizleme Adımları
- **Bölgesel Ayar (Locale):** Ondalık ayırıcı hatalarını önlemek için içe aktarma dili English (United States) olarak yapılandırıldı.
- **Yorum Satır Sonu Düzeltmesi (Reviews):** Yorum metinlerindeki satır kırılımlarının tabloyu bozmaması için Ignore quoted line breaks aktif edildi.
- **Posta Kodu Sıfırlarını Koruma (Customers, Sellers):** zip_code_prefix sütunlarının otomatik sayıya dönüşmesi engellenerek Text tipine çevrildi (01037 formatı korundu).
- **Teslimat Hesaplamaları (Orders):**
  - order_date: DateTime.Date([order_purchase_timestamp])
  - delivery_days: Duration.Days([order_delivered_customer_date] - [order_purchase_timestamp])
  - delay_days: Duration.Days(DateTime.Date([order_delivered_customer_date]) - DateTime.Date([order_estimated_delivery_date]))
  - delivery_status: delay_days > 0 ise "Late", <= 0 ise "On Time", boş ise "Not Delivered"
- **Kategori Çevirisi (Products):** Category Translation tablosu Left Outer Join ile Products tablosuna birleştirildi ve Enable load kapatılarak model şişkinliği önlendi.

---

## 2. Boyut ve Hesaplanmış Sütunlar (DAX)

### DimDate (Takvim Tablosu)
    DimDate = 
    ADDCOLUMNS (
        CALENDAR ( DATE ( 2016, 1, 1 ), DATE ( 2018, 12, 31 ) ),
        "Year", YEAR ( [Date] ),
        "Quarter", "Q" & QUARTER ( [Date] ),
        "Month Number", MONTH ( [Date] ),
        "Month", FORMAT ( [Date], "MMM", "en-US" ),
        "Year Month", FORMAT ( [Date], "yyyy-MM" ),
        "Weekday Number", WEEKDAY ( [Date], 2 ),
        "Weekday", FORMAT ( [Date], "ddd", "en-US" )
    )

### Orders[Delay Bucket] (Gecikme Kademeleri)
    Delay Bucket = 
    SWITCH (
        TRUE (),
        ISBLANK ( Orders[delay_days] ), "Not Delivered",
        Orders[delay_days] <= -10, "10+ days early",
        Orders[delay_days] <= -1, "1-9 days early",
        Orders[delay_days] = 0, "On the day",
        Orders[delay_days] <= 5, "1-5 days late",
        Orders[delay_days] <= 15, "6-15 days late",
        "15+ days late"
    )

---

## 3. DAX Ölçüleri (_Measures Tablosu)

### Sales (Satış)
- Total Revenue = SUM ( OrderItems[price] )
- Total Freight = SUM ( OrderItems[freight_value] )
- Total Orders = DISTINCTCOUNT ( OrderItems[order_id] )
- Items Sold = COUNTROWS ( OrderItems )
- Average Order Value = DIVIDE ( [Total Revenue], [Total Orders] )
- Active Sellers = DISTINCTCOUNT ( OrderItems[seller_id] )

### Customers (Müşteri)
- Total Customers = CALCULATE ( DISTINCTCOUNT ( Customers[customer_unique_id] ), OrderItems )
- Repeat Customers = COUNTROWS ( FILTER ( CALCULATETABLE ( VALUES ( Customers[customer_unique_id] ), OrderItems ), CALCULATE ( DISTINCTCOUNT ( OrderItems[order_id] ) ) > 1 ) )
- Repeat Customer Rate = DIVIDE ( [Repeat Customers], [Total Customers] )

### Delivery (Teslimat)
- Delivered Orders = CALCULATE ( COUNTROWS ( Orders ), Orders[delivery_status] <> "Not Delivered", OrderItems )
- Late Orders = CALCULATE ( COUNTROWS ( Orders ), Orders[delivery_status] = "Late", OrderItems )
- Late Delivery Rate = DIVIDE ( [Late Orders], [Delivered Orders] )
- Avg Delivery Days = AVERAGE ( Orders[delivery_days] )

### Reviews (Müşteri Memnuniyeti)
- Avg Review Score = CALCULATE ( AVERAGE ( Reviews[review_score] ), TREATAS ( VALUES ( OrderItems[order_id] ), Orders[order_id] ) )

### Time Intelligence (Zaman Zekâsı)
- Revenue LY = CALCULATE ( [Total Revenue], SAMEPERIODLASTYEAR ( DimDate[Date] ) )
- Revenue YoY % = DIVIDE ( [Total Revenue] - [Revenue LY], [Revenue LY] )
