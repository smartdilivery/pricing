# تحليل شامل لنظام حساب سعر التوصيل وخطة التعديل

## 📊 التحليل الحالي

### 1. **الموقع الرئيسي لحساب سعر التوصيل**
الحساب **موحد** في ملف واحد رئيسي:
- **`app/Services/EstimateOrderService.php`**
  - الدالة الرئيسية: `estimate($store, $data, $orderType = "by_store")`
  - تُستخدم في جميع أنواع الطلبات (متجر، تطبيق، سفاري)

### 2. **آلية الحساب الحالية**

#### أ. حساب سعر التوصيل (`delivery_price`):
```php
// 1. إذا كان هناك region_id
if ($store->region_id || $data['region_id']) {
    calculateRegionBasedDeliveryPrice($regionId, $distance, $store);
    // يستخدم: Region->delivery_price + (distance - min_distance) * price_per_km
}

// 2. إذا لم يكن هناك region، يُحسب حسب السوق:
if ($store->suwq_tr) {
    calculateDeliveryPriceForStoreTr($distance);
    // initial_price_tr + (distance - initial_distance_tr) * additional_price_tr
} else {
    calculateDeliveryPrice($distance);
    // initial_price + (distance - initial_distance) * additional_price
}

// 3. إذا كان الطلب سفاري، إضافة نسبة:
if ($orderType === 'safari') {
    delivery_price += delivery_price * (safari_orders_extra_delivery / 100);
}
```

#### ب. حساب رسوم المتجر (`store_fee`):
```php
store_fee = delivery_price * (taken_percentage_from_store / 100);
// النسبة من: store->setting->taken_percentage_from_store
// أو من settings("taken_percentage_from_delivery")
```

#### ج. توزيع السعر:
```php
delivery_fee_payed_by_store = delivery_price * (delivery_price_percentage / 100);
customer_payed_for_delivery = delivery_price - delivery_fee_payed_by_store;
driver_fee = store_fee; // حاليًا driver_fee = store_fee
```

### 3. **نقاط الاستخدام الحالية**
- ✅ `OrderController::estimate()` - تقدير السعر قبل الإنشاء
- ✅ `CreateOrderService::createOrder()` - إنشاء الطلب
- ✅ `ApplicationMakeOrder` - طلبات التطبيق
- ✅ `CustomerMakeSafariOrder` - طلبات السفاري
- ✅ `CustomerController::estimateDeliveryPrice()` - تقدير للعملاء

### 4. **الحقول في جدول Orders**
```sql
- total_price           -- إجمالي سعر الطلب (المنتجات)
- delivery_price        -- سعر التوصيل الكامل
- delivery_fee_payed_by_store -- الجزء الذي يدفعه المتجر
- driver_fee            -- رسوم السائق (حاليًا = store_fee)
- store_fee             -- رسوم المتجر (النسبة المأخوذة)
- distance              -- المسافة بالكيلومتر
```

---

## 🎯 المشاكل الحالية

### 1. **عدم الفصل بين سعر المتجر وسعر السائق**
- حاليًا: `driver_fee = store_fee`
- المشكلة: لا يوجد تمييز واضح بين ما يدفعه المتجر وما يحصل عليه السائق

### 2. **عدم دعم نظام التسعير الجديد**
- لا يوجد تكامل مع `StoreSubscription` و `StoreDistanceRateTable`
- لا يتم استخدام `PricingType` (per_order أو hourly)

### 3. **الخلط بين أنواع الأسعار**
- `delivery_price` = سعر التوصيل للعميل
- `store_fee` = رسوم النظام من المتجر
- `driver_fee` = ما يحصل عليه السائق (غير واضح)

---

## 🚀 الخطة الشاملة للتعديل

### المرحلة 1: إضافة حقول جديدة في جدول Orders

```php
// migration جديد
Schema::table('orders', function (Blueprint $table) {
    // فصل سعر التوصيل
    $table->decimal('store_delivery_cost', 11, 2)->default(0)->after('delivery_price')
        ->comment('تكلفة التوصيل التي يتحملها المتجر (من نظام التسعير)');
    
    $table->decimal('driver_delivery_earning', 11, 2)->default(0)->after('driver_fee')
        ->comment('مبلغ التوصيل الذي يحصل عليه السائق');
    
    // ربط مع نظام التسعير
    $table->foreignId('store_subscription_id')->nullable()->after('store_id')
        ->constrained('store_subscriptions')->nullOnDelete()
        ->comment('اشتراك التسعير المستخدم للطلب');
    
    $table->string('pricing_type', 20)->nullable()->after('store_subscription_id')
        ->comment('نوع التسعير المستخدم: per_order, hourly, legacy');
    
    $table->json('pricing_breakdown')->nullable()
        ->comment('تفصيل حساب السعر (للتوثيق)');
});
```

### المرحلة 2: إنشاء Service جديد - DeliveryPricingService

```php
<?php

namespace App\Services;

use App\Models\Order;
use App\Models\Store;
use App\PricingSystem\Models\StoreSubscription;
use App\PricingSystem\Services\StorePricingService;
use App\PricingSystem\Enums\PricingType;

/**
 * خدمة موحدة لحساب أسعار التوصيل
 * تدعم النظام القديم والنظام الجديد
 */
class DeliveryPricingService
{
    public function __construct(
        private EstimateOrderService $legacyService,
        private StorePricingService $newPricingService
    ) {}

    /**
     * حساب سعر التوصيل الشامل
     * 
     * @return array [
     *   'customer_delivery_price' => float,    // ما يدفعه العميل
     *   'store_delivery_cost' => float,        // ما يتحمله المتجر
     *   'driver_delivery_earning' => float,    // ما يحصل عليه السائق
     *   'store_fee' => float,                  // رسوم النظام
     *   'pricing_type' => string,              // per_order|hourly|legacy
     *   'subscription_id' => int|null,
     *   'breakdown' => array                   // تفصيل الحساب
     * ]
     */
    public function calculateDeliveryPricing(Store $store, array $orderData, string $orderType = 'by_store'): array
    {
        // 1. التحقق من وجود اشتراك فعّال
        $activeSubscription = $store->activeSubscription;
        
        if ($activeSubscription && $activeSubscription->status === 'active') {
            // استخدام نظام التسعير الجديد
            return $this->calculateWithNewPricing($store, $activeSubscription, $orderData, $orderType);
        }
        
        // استخدام النظام القديم
        return $this->calculateWithLegacyPricing($store, $orderData, $orderType);
    }
    
    /**
     * حساب بنظام التسعير الجديد
     */
    private function calculateWithNewPricing(
        Store $store,
        StoreSubscription $subscription,
        array $orderData,
        string $orderType
    ): array {
        $distance = $orderData['distance'] ?? 0;
        
        // 1. حساب سعر التوصيل للعميل (من النظام القديم - لا يتغير)
        $legacyEstimate = $this->legacyService->estimate($store, $orderData, $orderType);
        $customerDeliveryPrice = $legacyEstimate['delivery_price'];
        
        // 2. حساب تكلفة التوصيل للمتجر (من نظام التسعير الجديد)
        $mockOrder = (object)['distance' => $distance];
        $storeCostData = $this->newPricingService->calculateOrderPrice(
            $mockOrder, 
            $subscription, 
            $store
        );
        
        $storeDeliveryCost = $storeCostData['total_amount'];
        
        // 3. حساب مبلغ السائق
        $driverEarning = $this->calculateDriverEarning(
            $subscription->pricing_type,
            $storeDeliveryCost,
            $customerDeliveryPrice,
            $store
        );
        
        // 4. حساب رسوم النظام
        $storeFee = $storeDeliveryCost; // المتجر يدفع حسب اشتراكه
        
        return [
            'customer_delivery_price' => $customerDeliveryPrice,
            'store_delivery_cost' => $storeDeliveryCost,
            'driver_delivery_earning' => $driverEarning,
            'store_fee' => $storeFee,
            'delivery_fee_payed_by_store' => $legacyEstimate['delivery_fee_payed_by_store'],
            'pricing_type' => $subscription->pricing_type->value,
            'subscription_id' => $subscription->id,
            'breakdown' => [
                'subscription_plan' => $subscription->pricing_type->value,
                'distance' => $distance,
                'customer_pays' => $customerDeliveryPrice,
                'store_pays' => $storeDeliveryCost,
                'driver_receives' => $driverEarning,
                'system_fee' => $storeFee,
                'kdv_details' => $storeCostData,
            ],
        ];
    }
    
    /**
     * حساب بالنظام القديم (للمتاجر بدون اشتراك)
     */
    private function calculateWithLegacyPricing(
        Store $store,
        array $orderData,
        string $orderType
    ): array {
        $estimate = $this->legacyService->estimate($store, $orderData, $orderType);
        
        return [
            'customer_delivery_price' => $estimate['delivery_price'],
            'store_delivery_cost' => $estimate['store_fee'],
            'driver_delivery_earning' => $estimate['driver_fee'],
            'store_fee' => $estimate['store_fee'],
            'delivery_fee_payed_by_store' => $estimate['delivery_fee_payed_by_store'],
            'pricing_type' => 'legacy',
            'subscription_id' => null,
            'breakdown' => [
                'legacy_calculation' => true,
                'distance' => $orderData['distance'] ?? 0,
            ],
        ];
    }
    
    /**
     * حساب مبلغ السائق حسب نوع التسعير
     */
    private function calculateDriverEarning(
        PricingType $pricingType,
        float $storeDeliveryCost,
        float $customerDeliveryPrice,
        Store $store
    ): float {
        if ($pricingType === PricingType::PER_ORDER) {
            // في نظام بالطلب: السائق يحصل على نسبة من سعر العميل
            $driverPercentage = settings('driver_percentage_taken_from_delivery') ?? 70;
            return $customerDeliveryPrice * ($driverPercentage / 100);
        }
        
        if ($pricingType === PricingType::HOURLY) {
            // في نظام الساعات: السائق يحصل على المبلغ الثابت من جدول الورديات
            // هذا يتم تحديده في DriverShift->per_order_amount
            // نعيد المبلغ من الاشتراك كـ fallback
            return $storeDeliveryCost;
        }
        
        return 0;
    }
}
```

### المرحلة 3: تعديل CreateOrderService

```php
// في CreateOrderService::createOrder()

public function createOrder($data, $store, $orderType = "by_store")
{
    // استخدام الخدمة الجديدة
    $pricingService = app(DeliveryPricingService::class);
    $pricingResult = $pricingService->calculateDeliveryPricing($store, $data, $orderType);
    
    // الحصول على التقدير القديم للحقول الأخرى
    $legacyEstimate = $this->estimateService->estimate($store, $data, $orderType);
    
    // دمج النتائج
    $result = array_merge($legacyEstimate, [
        'delivery_price' => $pricingResult['customer_delivery_price'],
        'store_delivery_cost' => $pricingResult['store_delivery_cost'],
        'driver_delivery_earning' => $pricingResult['driver_delivery_earning'],
        'store_fee' => $pricingResult['store_fee'],
        'pricing_type' => $pricingResult['pricing_type'],
        'subscription_id' => $pricingResult['subscription_id'],
        'pricing_breakdown' => $pricingResult['breakdown'],
    ]);
    
    if (!$store->suwq_tr) {
        $this->storeClient($store, $result);
    }
    
    $final = $this->prepareData($store, $result, $orderType);
    $order = $this->orderRepository->store($final);
    
    // ... بقية الكود
}
```

### المرحلة 4: تحديث prepareData

```php
public function prepareData($store, $result, $orderType = "by_store")
{
    $lastOrder = $store->orders()->latest()->first();
    
    $insert = Arr::except($result, [
        "expected_time", "code", "price", "original_price", 
        "customer_payed", "store_profit", "total_with_delivery",
        "customer_payed_for_delivery", "breakdown"
    ]);
    
    // إضافة الحقول الجديدة
    $insert['store_delivery_cost'] = $result['store_delivery_cost'] ?? 0;
    $insert['driver_delivery_earning'] = $result['driver_delivery_earning'] ?? 0;
    $insert['store_subscription_id'] = $result['subscription_id'] ?? null;
    $insert['pricing_type'] = $result['pricing_type'] ?? 'legacy';
    $insert['pricing_breakdown'] = json_encode($result['breakdown'] ?? []);
    
    // ... بقية الكود كما هو
}
```

---

## 📋 جدول المقارنة: القديم vs الجديد

| البند | النظام القديم | النظام الجديد (مع اشتراك) |
|------|---------------|--------------------------|
| **سعر العميل** | settings/region based | نفس الطريقة (لا يتغير) |
| **تكلفة المتجر** | `store_fee` (نسبة ثابتة) | `store_delivery_cost` (حسب الاشتراك) |
| **مبلغ السائق** | `driver_fee = store_fee` | `driver_delivery_earning` (منفصل) |
| **الحساب** | موحد للجميع | مخصص حسب كل متجر |
| **التوثيق** | غير موجود | `pricing_breakdown` JSON |

---

## ⚡ خطوات التنفيذ

### 1. **قاعدة البيانات** (يوم 1)
- [ ] إنشاء migration لإضافة الحقول الجديدة
- [ ] تشغيل migration على بيئة التطوير
- [ ] اختبار التوافق العكسي

### 2. **إنشاء Service** (يوم 2-3)
- [ ] إنشاء `DeliveryPricingService`
- [ ] تنفيذ `calculateWithNewPricing()`
- [ ] تنفيذ `calculateWithLegacyPricing()`
- [ ] تنفيذ `calculateDriverEarning()`
- [ ] كتابة Unit Tests

### 3. **تعديل Services الموجودة** (يوم 4)
- [ ] تعديل `CreateOrderService`
- [ ] تعديل `ApplicationMakeOrder`
- [ ] تعديل `CustomerMakeSafariOrder`
- [ ] التأكد من التوافق الخلفي

### 4. **اختبار شامل** (يوم 5)
- [ ] اختبار المتاجر مع اشتراك per_order
- [ ] اختبار المتاجر مع اشتراك hourly
- [ ] اختبار المتاجر بدون اشتراك (legacy)
- [ ] اختبار طلبات سفاري
- [ ] اختبار الحسابات المالية

### 5. **النشر** (يوم 6)
- [ ] مراجعة الكود
- [ ] نشر على staging
- [ ] اختبار شامل على staging
- [ ] نشر على production
- [ ] مراقبة الأخطاء

---

## 🔍 نقاط مهمة

### ✅ المميزات
1. **فصل واضح** بين سعر المتجر وسعر السائق
2. **دعم كامل** لنظام التسعير الجديد
3. **توافق خلفي** مع النظام القديم
4. **توثيق شامل** لكل حساب في `pricing_breakdown`
5. **سهولة الصيانة** - كل شيء في مكان واحد

### ⚠️ التحديات المحتملة
1. **البيانات القديمة**: الطلبات القديمة لن يكون لها الحقول الجديدة
2. **التقارير**: قد تحتاج تحديث للتعامل مع الحقول الجديدة
3. **الواجهات**: Frontend قد يحتاج تحديث لعرض التفاصيل الجديدة

### 🎯 الحل للتحديات
1. استخدام `->default(0)` في Migration
2. إضافة Accessor في Model للتوافق
3. إنشاء API endpoints جديدة للتفاصيل

---

## 📝 مثال عملي

### طلب من متجر مع اشتراك per_order:
```json
{
  "distance": 5.2,
  "customer_delivery_price": 50.00,    // ما يدفعه العميل (حسب النظام القديم)
  "store_delivery_cost": 35.00,        // ما يدفعه المتجر (من جدول الأسعار)
  "driver_delivery_earning": 35.00,    // ما يحصل عليه السائق (70% من 50)
  "store_fee": 35.00,                  // رسوم النظام = تكلفة المتجر
  "pricing_type": "per_order",
  "subscription_id": 123
}
```

### طلب من متجر مع اشتراك hourly:
```json
{
  "distance": 5.2,
  "customer_delivery_price": 50.00,    // ما يدفعه العميل
  "store_delivery_cost": 10.00,        // مبلغ ثابت لكل طلب من الاشتراك
  "driver_delivery_earning": 10.00,    // السائق يحصل على المبلغ الثابت
  "store_fee": 10.00,
  "pricing_type": "hourly",
  "subscription_id": 124
}
```

### طلب من متجر بدون اشتراك (legacy):
```json
{
  "distance": 5.2,
  "customer_delivery_price": 50.00,
  "store_delivery_cost": 15.00,        // حسب النسبة القديمة
  "driver_delivery_earning": 15.00,    // = store_fee
  "store_fee": 15.00,
  "pricing_type": "legacy",
  "subscription_id": null
}
```

---

## 🎓 الخلاصة

هذه الخطة تضمن:
- ✅ **توافق كامل** مع النظام القديم
- ✅ **دعم كامل** للنظام الجديد
- ✅ **فصل واضح** بين أسعار المتجر والسائق
- ✅ **سهولة الصيانة** والتطوير المستقبلي
- ✅ **توثيق شامل** لكل عملية حسابية
