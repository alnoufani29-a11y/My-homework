<?php
/*******************************************************
 * Bank System - Single File PHP App
 * ملف واحد يشمل: إنشاء/اتصال قاعدة البيانات، إنشاء الجداول،
 * إضافة حساب، تحويل الأموال، عرض الحسابات، وتسجيل العمليات
 * باستخدام PDO ومعاملات (Transactions) لضمان الاتساق.
 *******************************************************/

/*-------------------------*
 | إعدادات الاتصال بقاعدة البيانات
 *-------------------------*/
$dbHost = 'localhost';
$dbUser = 'root';
$dbPass = '';           // عدّل كلمة المرور إن لزم
$dbName = 'bank_system';// اسم قاعدة البيانات المطلوب

/*-------------------------*
 | دوال مساعدة
 *-------------------------*/

/**
 * إنشاء اتصال PDO بقاعدة البيانات (مع محاولة إنشاء قاعدة البيانات إن لم تكن موجودة)
 */
function getPdo($dbHost, $dbUser, $dbPass, $dbName)
{
    // حاول الاتصال بدون تحديد قاعدة البيانات أولًا لإنشاءها إن لم تكن موجودة
    $pdo = new PDO("mysql:host={$dbHost};charset=utf8mb4", $dbUser, $dbPass, [
        PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
    ]);

    // أنشئ قاعدة البيانات إذا لم تكن موجودة
    $pdo->exec("CREATE DATABASE IF NOT EXISTS `{$dbName}` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci");

    // اتصل بقاعدة البيانات المطلوبة
    $pdo = new PDO("mysql:host={$dbHost};dbname={$dbName};charset=utf8mb4", $dbUser, $dbPass, [
        PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
    ]);

    return $pdo;
}

/**
 * إنشاء الجداول إذا لم تكن موجودة
 */
function ensureSchema(PDO $pdo)
{
    // جدول الحسابات
    $pdo->exec("
        CREATE TABLE IF NOT EXISTS accounts (
            id INT AUTO_INCREMENT PRIMARY KEY,
            name VARCHAR(100) NOT NULL,
            balance DECIMAL(12,2) NOT NULL DEFAULT 0.00,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        ) ENGINE=InnoDB;
    ");

    // جدول التحويلات
    $pdo->exec("
        CREATE TABLE IF NOT EXISTS transactions (
            id INT AUTO_INCREMENT PRIMARY KEY,
            from_account INT NULL,
            to_account INT NULL,
            amount DECIMAL(12,2) NOT NULL,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (from_account) REFERENCES accounts(id) ON DELETE SET NULL,
            FOREIGN KEY (to_account) REFERENCES accounts(id) ON DELETE SET NULL
        ) ENGINE=InnoDB;
    ");
}

/**
 * تنظيف مدخلات نصية
 */
function cleanStr($v) {
    return trim((string)$v);
}

/**
 * إرجاع رسالة منسقة
 */
function flash($type, $msg) {
    return ['type' => $type, 'msg' => $msg];
}

/*-------------------------*
 | بدء التنفيذ
 *-------------------------*/
$messages = [];

try {
    $pdo = getPdo($dbHost, $dbUser, $dbPass, $dbName);
    ensureSchema($pdo);

    // معالجة النماذج
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {

        // إضافة حساب جديد
        if (isset($_POST['action']) && $_POST['action'] === 'add_account') {
            $name    = cleanStr($_POST['name'] ?? '');
            $balance = (float)($_POST['balance'] ?? 0);

            if ($name === '') {
                $messages[] = flash('error', 'الاسم مطلوب.');
            } elseif ($balance < 0) {
                $messages[] = flash('error', 'الرصيد لا يمكن أن يكون سالبًا.');
            } else {
                $stmt = $pdo->prepare("INSERT INTO accounts (name, balance) VALUES (?, ?)");
                $stmt->execute([$name, $balance]);
                $messages[] = flash('success', 'تم إضافة الحساب بنجاح.');
            }
        }

        // تحويل الأموال بين الحسابات
        if (isset($_POST['action']) && $_POST['action'] === 'transfer') {
            $fromId = (int)($_POST['from_id'] ?? 0);
            $toId   = (int)($_POST['to_id'] ?? 0);
            $amount = (float)($_POST['amount'] ?? 0);

            if ($fromId <= 0 || $toId <= 0) {
                $messages[] = flash('error', 'يجب اختيار حسابي التحويل.');
            } elseif ($fromId === $toId) {
                $messages[] = flash('error', 'لا يمكن التحويل لنفس الحساب.');
            } elseif ($amount <= 0) {
                $messages[] = flash('error', 'المبلغ يجب أن يكون أكبر من صفر.');
            } else {
                // استخدم معاملة لضمان الاتساق
                $pdo->beginTransaction();
                try {
                    // جلب رصيد الحساب المحوِّل
                    $stmt = $pdo->prepare("SELECT id, balance FROM accounts WHERE id = ?");
                    $stmt->execute([$fromId]);
                    $fromAcc = $stmt->fetch();

                    // جلب رصيد الحساب المستلِم
                    $stmt = $pdo->prepare("SELECT id, balance FROM accounts WHERE id = ?");
                    $stmt->execute([$toId]);
                    $toAcc = $stmt->fetch();

                    if (!$fromAcc || !$toAcc) {
                        throw new Exception('أحد الحسابات غير موجود.');
                    }

                    if ($fromAcc['balance'] < $amount) {
                        throw new Exception('رصيد الحساب المحوِّل غير كافٍ.');
                    }

                    // خصم من الحساب المحوِّل
                    $stmt = $pdo->prepare("UPDATE accounts SET balance = balance - ? WHERE id = ?");
                    $stmt->execute([$amount, $fromId]);

                    // إضافة إلى الحساب المستلِم
                    $stmt = $pdo->prepare("UPDATE accounts SET balance = balance + ? WHERE id = ?");
                    $stmt->execute([$amount, $toId]);

                    // تسجيل العملية
                    $stmt = $pdo->prepare("INSERT INTO transactions (from_account, to_account, amount) VALUES (?, ?, ?)");
                    $stmt->execute([$fromId, $toId, $amount]);

                    $pdo->commit();
                    $messages[] = flash('success', 'تمت عملية التحويل بنجاح.');
                } catch (Exception $e) {
                    $pdo->rollBack();
                    $messages[] = flash('error', 'فشلت عملية التحويل: ' . $e->getMessage());
                }
            }
        }
    }

    // جلب الحسابات للعرض والاختيار
    $accounts = $pdo->query("SELECT id, name, balance, created_at FROM accounts ORDER BY id DESC")->fetchAll();

    // جلب آخر 10 عمليات
    $transactions = $pdo->query("
        SELECT t.id, t.amount, t.created_at,
               fa.name AS from_name, ta.name AS to_name
        FROM transactions t
        LEFT JOIN accounts fa ON fa.id = t.from_account
        LEFT JOIN accounts ta ON ta.id = t.to_account
        ORDER BY t.id DESC
        LIMIT 10
    ")->fetchAll();

} catch (Throwable $ex) {
    $messages[] = flash('error', 'خطأ اتصال بقاعدة البيانات: ' . $ex->getMessage());
}

?>
<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="utf-8">
    <title>نظام بنكي تجريبي - ملف واحد</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <style>
        /* CSS مدمج - بسيط ونظيف */
        :root {
            --bg: #0f172a;
            --card: #111827;
            --text: #e5e7eb;
            --muted: #9ca3af;
            --accent: #22c55e;
            --accent2: #3b82f6;
            --danger: #ef4444;
            --border: #1f2937;
        }
        * { box-sizing: border-box; font-family: "Segoe UI", Tahoma, sans-serif; }
        body { margin: 0; background: var(--bg); color: var(--text); }
        .container { max-width: 1100px; margin: 40px auto; padding: 0 16px; }
        header { margin-bottom: 24px; }
        h1 { margin: 0 0 8px; font-size: 24px; }
        .sub { color: var(--muted); font-size: 14px; }
        .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; }
        .card { background: var(--card); border: 1px solid var(--border); border-radius: 12px; padding: 16px; }
        .card h2 { margin: 0 0 12px; font-size: 18px; }
        label { display: block; margin-bottom: 6px; font-size: 13px; color: var(--muted); }
        input[type="text"], input[type="number"], select {
            width: 100%; padding: 10px; border: 1px solid var(--border);
            border-radius: 8px; background: #0b1220; color: var(--text);
        }
        .row { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }
        .btn {
            display: inline-block; padding: 10px 14px; border-radius: 8px;
            border: none; cursor: pointer; font-weight: 600;
        }
        .btn-primary { background: var(--accent2); color: white; }
        .btn-success { background: var(--accent); color: black; }
        .btn-danger { background: var(--danger); color: white; }
        .messages { margin-bottom: 16px; }
        .msg { padding: 10px 12px; border-radius: 8px; font-size: 14px; margin-bottom: 8px; }
        .msg.success { background: rgba(34,197,94,0.15); border: 1px solid rgba(34,197,94,0.3); }
        .msg.error { background: rgba(239,68,68,0.15); border: 1px solid rgba(239,68,68,0.3); }
        table { width: 100%; border-collapse: collapse; }
        th, td { padding: 10px; border-bottom: 1px solid var(--border); font-size: 14px; text-align: left; }
        th { color: var(--muted); font-weight: 600; }
        .muted { color: var(--muted); }
        footer { margin-top: 24px; color: var(--muted); font-size: 13px; text-align: center; }
        @media (max-width: 900px) { .grid { grid-template-columns: 1fr; } .row { grid-template-columns: 1fr; } }
    </style>
</head>
<body>
<div class="container">
    <header>
        <h1>🏦 نظام بنكي تجريبي (ملف واحد)</h1>
        <div class="sub">إضافة حساب، تحويل الأموال، عرض الحسابات، وتسجيل العمليات — كل ذلك في صفحة واحدة.</div>
    </header>

    <div class="messages">
        <?php foreach ($messages as $m): ?>
            <div class="msg <?= htmlspecialchars($m['type']) ?>"><?= htmlspecialchars($m['msg']) ?></div>
        <?php endforeach; ?>
    </div>

    <div class="grid">
        <!-- إضافة حساب جديد -->
        <div class="card">
            <h2>إضافة حساب جديد</h2>
            <form method="post">
                <input type="hidden" name="action" value="add_account">
                <label>اسم الحساب</label>
                <input type="text" name="name" placeholder="مثال: حساب زيد" required>
                <div class="row" style="margin-top:8px;">
                    <div>
                        <label>رصيد ابتدائي</label>
                        <input type="number" name="balance" min="0" step="0.01" placeholder="0.00" value="0.00">
                    </div>
                </div>
                <div style="margin-top:12px;">
                    <button class="btn btn-success" type="submit">إضافة الحساب</button>
                </div>
            </form>
        </div>

        <!-- تحويل الأموال -->
        <div class="card">
            <h2>تحويل الأموال</h2>
            <form method="post" onsubmit="return confirm('تأكيد: هل تريد تنفيذ عملية التحويل؟');">
                <input type="hidden" name="action" value="transfer">
                <div class="row">
                    <div>
                        <label>من الحساب</label>
                        <select name="from_id" required>
                            <option value="">اختر الحساب</option>
                            <?php foreach ($accounts as $acc): ?>
                                <option value="<?= (int)$acc['id'] ?>">
                                    <?= htmlspecialchars($acc['name']) ?> — رصيد: <?= number_format((float)$acc['balance'], 2) ?>
                                </option>
                            <?php endforeach; ?>
                        </select>
                    </div>
                    <div>
                        <label>إلى الحساب</label>
                        <select name="to_id" required>
                            <option value="">اختر الحساب</option>
                            <?php foreach ($accounts as $acc): ?>
                                <option value="<?= (int)$acc['id'] ?>">
                                    <?= htmlspecialchars($acc['name']) ?> — رصيد: <?= number_format((float)$acc['balance'], 2) ?>
                                </option>
                            <?php endforeach; ?>
                        </select>
                    </div>
                </div>
                <div class="row" style="margin-top:8px;">
                    <div>
                        <label>المبلغ</label>
                        <input type="number" name="amount" min="0.01" step="0.01" placeholder="0.00" required>
                    </div>
                </div>
                <div style="margin-top:12px;">
                    <button class="btn btn-primary" type="submit">تنفيذ التحويل</button>
                </div>
            </form>
        </div>
    </div>

    <!-- عرض الحسابات -->
    <div class="card" style="margin-top:16px;">
        <h2>قائمة الحسابات الحالية</h2>
        <table>
            <thead>
            <tr>
                <th>#</th>
                <th>الاسم</th>
                <th>الرصيد</th>
                <th>تاريخ الإنشاء</th>
            </tr>
            </thead>
            <tbody>
            <?php if (!empty($accounts)): ?>
                <?php foreach ($accounts as $acc): ?>
                    <tr>
                        <td><?= (int)$acc['id'] ?></td>
                        <td><?= htmlspecialchars($acc['name']) ?></td>
                        <td><?= number_format((float)$acc['balance'], 2) ?></td>
                        <td class="muted"><?= htmlspecialchars($acc['created_at']) ?></td>
                    </tr>
                <?php endforeach; ?>
            <?php else: ?>
                <tr><td colspan="4" class="muted">لا توجد حسابات بعد.</td></tr>
            <?php endif; ?>
            </tbody>
        </table>
    </div>

    <!-- عرض آخر العمليات -->
    <div class="card" style="margin-top:16px;">
        <h2>آخر العمليات</h2>
        <table>
            <thead>
            <tr>
                <th>#</th>
                <th>من</th>
                <th>إلى</th>
                <th>المبلغ</th>
                <th>التاريخ</th>
            </tr>
            </thead>
            <tbody>
            <?php if (!empty($transactions)): ?>
                <?php foreach ($transactions as $t): ?>
                    <tr>
                        <td><?= (int)$t['id'] ?></td>
                        <td><?= htmlspecialchars($t['from_name'] ?? '—') ?></td>
                        <td><?= htmlspecialchars($t['to_name'] ?? '—') ?></td>
                        <td><?= number_format((float)$t['amount'], 2) ?></td>
                        <td class="muted"><?= htmlspecialchars($t['created_at']) ?></td>
                    </tr>
                <?php endforeach; ?>
            <?php else: ?>
                <tr><td colspan="5" class="muted">لا توجد عمليات حتى الآن.</td></tr>
            <?php endif; ?>
            </tbody>
        </table>
    </div>

    <footer>
        تم البناء بملف واحد باستخدام PHP وPDO وInnoDB لضمان الاتساق في التحويلات.
    </footer>
</div>
</body>
</html>
