1-Accounts table
CREATE TABLE accounts (
id INT PRIMARY KEY AUTO_INCREMENT,
name VARCHAR(100),
balance DECIMAL(10,2)
);

2-transactions table
CREATE TABLE transactions (
id INT PRIMARY KEY AUTO_INCREMENT,
from_account INT,
to_account INT,
amount DECIMAL(10,2),
created_at DATETIME
);

3-code PHP
<?php
// الاتصال بقاعدة البيانات
$pdo = new PDO("mysql:host=localhost;dbname=bank", "root", "");
$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

// بيانات التحويل
$fromAccount = 1;
$toAccount = 2;
$amount = 100;

// نبدأ المعاملة (Transaction)
$pdo->beginTransaction();

try {

// 1️⃣ خصم الرصيد من الحساب الأول
$stmt = $pdo->prepare(
"UPDATE accounts 
SET balance = balance - :amount 
WHERE id = :id AND balance >= :amount"
);
$stmt->execute([
':amount' => $amount,
':id' => $fromAccount
]);

// التحقق من نجاح الخصم
if ($stmt->rowCount() == 0) {
throw new Exception("الرصيد غير كافٍ");
}

// 2️⃣ إيداع الرصيد في الحساب الثاني
$stmt = $pdo->prepare(
"UPDATE accounts 
SET balance = balance + :amount 
WHERE id = :id"
);
$stmt->execute([
':amount' => $amount,
':id' => $toAccount
]);

// 3️⃣ تسجيل العملية في جدول الحركات
$stmt = $pdo->prepare(
"INSERT INTO transactions 
(from_account, to_account, amount, created_at)
VALUES (:from, :to, :amount, NOW())"
);
$stmt->execute([
':from' => $fromAccount,
':to' => $toAccount,
':amount' => $amount
]);

// ✅ نجاح جميع العمليات
$pdo->commit();
echo "تم التحويل بنجاح";

} catch (Exception $e) {

// ❌ في حال حدوث خطأ
$pdo->rollBack();
echo "فشل التحويل: " . $e->getMessage();
}

//شكرا 🌹chat gpt