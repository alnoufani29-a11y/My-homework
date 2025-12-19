طبعا يا استاذ تم الاستعانة ChatGpt
🗄️ طريقة الاتصال بقاعدة البيانات في لغة PHP

تُستخدم قواعد البيانات لتخزين واسترجاع البيانات، وأكثر قاعدة بيانات تُستخدم مع PHP هي MySQL.
يوجد طريقتان أساسيتان للاتصال بقاعدة البيانات في PHP.


---

1️⃣ الاتصال باستخدام MySQLi

(MySQL Improved)

🔹 مميزاتها

سهلة للمبتدئين

تدعم البرمجة الإجرائية (Procedural) والكائنية (OOP)

آمنة عند استخدام Prepared Statements



---

🔸 الاتصال بقاعدة البيانات (OOP)

$conn = new mysqli("localhost", "username", "password", "database");

if ($conn->connect_error) {
    die("فشل الاتصال: " . $conn->connect_error);
}

echo "تم الاتصال بنجاح";


---

🔸 الاتصال بقاعدة البيانات (Procedural)

$conn = mysqli_connect("localhost", "username", "password", "database");

if (!$conn) {
    die("فشل الاتصال: " . mysqli_connect_error());
}


---

🔸 إغلاق الاتصال

$conn->close();


---

2️⃣ الاتصال باستخدام PDO

(PHP Data Objects) ⭐ الأفضل

🔹 مميزاتها

أكثر أمانًا

تدعم أكثر من نوع قاعدة بيانات (MySQL – PostgreSQL – SQLite)

تمنع SQL Injection

احترافية ومستخدمة في المشاريع الكبيرة



---

🔸 الاتصال بقاعدة البيانات

try {
    $conn = new PDO("mysql:host=localhost;dbname=database", "username", "password");
    $conn->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    echo "تم الاتصال بنجاح";
} catch (PDOException $e) {
    echo "فشل الاتصال: " . $e->getMessage();
}


---

🔸 إغلاق الاتصال

$conn = null;


---

3️⃣ تنفيذ استعلام SQL (MySQLi)

🔹 إدخال بيانات

$sql = "INSERT INTO users (name, email) VALUES ('Ali', 'ali@email.com')";
$conn->query($sql);


---

🔹 جلب بيانات

$result = $conn->query("SELECT * FROM users");

while ($row = $result->fetch_assoc()) {
    echo $row["name"];
}


---

4️⃣ تنفيذ استعلام SQL (PDO)

🔹 إدخال بيانات آمن

$stmt = $conn->prepare("INSERT INTO users (name, email) VALUES (?, ?)");
$stmt->execute(["Ali", "ali@email.com"]);


---

🔹 جلب بيانات

$stmt = $conn->prepare("SELECT * FROM users");
$stmt->execute();

$data = $stmt->fetchAll();


---

5️⃣ Prepared Statements (مهم جدًا) 🔐

تحمي من SQL Injection

$stmt = $conn->prepare("SELECT * FROM users WHERE id = ?");
$stmt->execute([$id]);


---

6️⃣ مقارنة بين MySQLi و PDO

المقارنة	MySQLi	PDO

دعم قواعد متعددة	❌	✅
الأمان	جيد	ممتاز
السهولة	أسهل	احترافي
يُنصح به	مشاريع صغيرة	مشاريع كبيرة



---

📌 خلاصة البحث

MySQLi: مناسب للمبتدئين

PDO: الأفضل والأكثر أمانًا

استخدم Prepared Statements دائمًا

لا تكتب بيانات الدخول داخل الكود النهائي
