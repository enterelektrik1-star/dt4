<?php
declare(strict_types=1);
session_start();
require __DIR__ . '/config.php';

if (!is_dir(STORAGE_PATH)) mkdir(STORAGE_PATH, 0755, true);
$root = realpath(STORAGE_PATH);
if ($root === false) die('Folder penyimpanan tidak tersedia.');

function e($v): string { return htmlspecialchars((string)$v, ENT_QUOTES, 'UTF-8'); }
function logged(): bool { return !empty($_SESSION['fm_logged']); }
function csrf(): string {
    if (empty($_SESSION['csrf'])) $_SESSION['csrf'] = bin2hex(random_bytes(32));
    return $_SESSION['csrf'];
}
function check_csrf(): void {
    if (!hash_equals($_SESSION['csrf'] ?? '', $_POST['csrf'] ?? '')) {
        http_response_code(403); exit('Permintaan tidak valid (CSRF).');
    }
}
function clean_rel(string $path): string {
    $path = str_replace('\\', '/', trim($path));
    $parts = [];
    foreach (explode('/', $path) as $part) {
        if ($part === '' || $part === '.') continue;
        if ($part === '..' || strpos($part, "\0") !== false) {
            http_response_code(400); exit('Path tidak diizinkan.');
        }
        $parts[] = $part;
    }
    return implode('/', $parts);
}
function target_path(string $rel, bool $mustExist = true): string {
    global $root;
    $rel = clean_rel($rel);
    $candidate = $root . ($rel !== '' ? DIRECTORY_SEPARATOR . str_replace('/', DIRECTORY_SEPARATOR, $rel) : '');
    if ($mustExist) {
        $real = realpath($candidate);
        if ($real === false || ($real !== $root && strpos($real, $root . DIRECTORY_SEPARATOR) !== 0)) {
            http_response_code(404); exit('File atau folder tidak ditemukan.');
        }
        return $real;
    }
    $parent = realpath(dirname($candidate));
    if ($parent === false || ($parent !== $root && strpos($parent, $root . DIRECTORY_SEPARATOR) !== 0)) {
        http_response_code(400); exit('Lokasi tidak diizinkan.');
    }
    return $candidate;
}
function flash(string $msg, string $type='success'): void { $_SESSION['flash'] = [$msg, $type]; }
function redirect_to(string $path=''): never {
    header('Location: index.php?path=' . rawurlencode($path)); exit;
}
function mime_type(string $file): string {
    if (function_exists('finfo_open')) {
        $f = finfo_open(FILEINFO_MIME_TYPE);
        if ($f) { $m = finfo_file($f, $file); finfo_close($f); if ($m) return $m; }
    }
    return 'application/octet-stream';
}

if (isset($_GET['logout'])) { session_destroy(); header('Location: index.php'); exit; }

// Login diproses terpisah; halaman utama tetap dapat diakses publik.
if (!logged() && $_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['login'])) {
    if (hash_equals(ADMIN_USERNAME, (string)($_POST['username'] ?? '')) &&
        hash_equals(ADMIN_PASSWORD, (string)($_POST['password'] ?? ''))) {
        session_regenerate_id(true);
        $_SESSION['fm_logged'] = true;
        csrf();
        header('Location: index.php');
        exit;
    }
    $_SESSION['login_error'] = 'Username atau password salah.';
    header('Location: index.php?login=1');
    exit;
}

// Form login hanya ditampilkan jika pengunjung memilih menu Login.
if (!logged() && isset($_GET['login'])) {
    $error = $_SESSION['login_error'] ?? '';
    unset($_SESSION['login_error']);
    ?><!doctype html><html lang="id"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1"><title>Login - <?=e(APP_NAME)?></title><link rel="stylesheet" href="assets/style.css"></head>
    <body class="login-body"><form class="login-card" method="post"><div class="brand-icon">▦</div><h1><?=e(APP_NAME)?></h1><p>Masuk untuk mengelola file dan folder</p>
    <?php if($error): ?><div class="alert error"><?=e($error)?></div><?php endif; ?>
    <label>Username</label><input name="username" required autocomplete="username" autofocus>
    <label>Password</label><input type="password" name="password" required autocomplete="current-password">
    <button class="btn primary full" name="login" value="1">Masuk</button>
    <p><a href="index.php">← Kembali ke file sharing</a></p>
    </form></body></html><?php
    exit;
}

$cwd = clean_rel((string)($_GET['path'] ?? ''));
$current = target_path($cwd);
if (!is_dir($current)) { $cwd = ''; $current = $root; }

if (isset($_GET['action']) && in_array($_GET['action'], ['download','preview'], true)) {
    $rel = clean_rel((string)($_GET['file'] ?? ''));
    $file = target_path($rel);
    if (!is_file($file)) { http_response_code(404); exit('Bukan file.'); }
    $name = basename($file); $mime = mime_type($file);
    if ($_GET['action'] === 'download') {
        header('Content-Type: application/octet-stream');
        header('Content-Disposition: attachment; filename="' . rawurlencode($name) . '"');
    } else {
        $allowed = ['image/jpeg','image/png','image/gif','image/webp','image/svg+xml','application/pdf','text/plain','text/csv','text/html','audio/mpeg','audio/ogg','audio/wav','video/mp4','video/webm'];
        if (!in_array($mime, $allowed, true)) { http_response_code(415); exit('Preview tidak tersedia untuk tipe file ini. Silakan download.'); }
        // HTML/SVG ditampilkan sebagai teks agar tidak menjalankan skrip dari file upload.
        if (in_array($mime, ['text/html','image/svg+xml'], true)) $mime = 'text/plain; charset=utf-8';
        header('Content-Type: ' . $mime);
        header('Content-Disposition: inline; filename="' . rawurlencode($name) . '"');
        header('X-Content-Type-Options: nosniff');
        header("Content-Security-Policy: default-src 'none'; img-src data:; media-src 'self'; style-src 'unsafe-inline'");
    }
    header('Content-Length: ' . filesize($file));
    readfile($file); exit;
}

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    if (!logged()) {
        http_response_code(403);
        exit('Akses ditolak. Silakan login untuk mengelola file.');
    }
    check_csrf();
    $action = $_POST['do'] ?? '';
    try {
        if ($action === 'mkdir') {
            $name = trim((string)$_POST['name']);
            if ($name === '' || $name === '.' || $name === '..' || preg_match('~[\\\\/:*?"<>|]~', $name)) throw new RuntimeException('Nama folder tidak valid.');
            $dest = target_path($cwd, true) . DIRECTORY_SEPARATOR . $name;
            if (file_exists($dest)) throw new RuntimeException('Nama sudah digunakan.');
            if (!mkdir($dest, 0755)) throw new RuntimeException('Gagal membuat folder.');
            flash('Folder berhasil dibuat.');
        } elseif ($action === 'upload') {
            if (empty($_FILES['upload_files'])) throw new RuntimeException('Pilih file terlebih dahulu.');
            $count = 0;
            foreach ($_FILES['upload_files']['name'] as $i => $original) {
                if ($_FILES['upload_files']['error'][$i] === UPLOAD_ERR_NO_FILE) continue;
                if ($_FILES['upload_files']['error'][$i] !== UPLOAD_ERR_OK) throw new RuntimeException('Upload gagal: ' . $original);
                if ($_FILES['upload_files']['size'][$i] > MAX_UPLOAD_BYTES) throw new RuntimeException('Ukuran file melebihi batas 50 MB.');
                $name = basename(str_replace('\\', '/', (string)$original));
                if ($name === '' || $name[0] === '.') throw new RuntimeException('Nama file tersembunyi atau tidak valid.');
                $dest = target_path($cwd, true) . DIRECTORY_SEPARATOR . $name;
                if (file_exists($dest)) throw new RuntimeException('File "' . $name . '" sudah ada.');
                if (!move_uploaded_file($_FILES['upload_files']['tmp_name'][$i], $dest)) throw new RuntimeException('Tidak dapat menyimpan file.');
                $count++;
            }
            flash($count . ' file berhasil diunggah.');
        } elseif ($action === 'rename') {
            $old = clean_rel((string)$_POST['old']);
            $item = target_path($old);
            $new = trim((string)$_POST['new']);
            if ($new === '' || $new === '.' || $new === '..' || preg_match('~[\\\\/:*?"<>|]~', $new)) throw new RuntimeException('Nama baru tidak valid.');
            $dest = dirname($item) . DIRECTORY_SEPARATOR . $new;
            if (file_exists($dest)) throw new RuntimeException('Nama tujuan sudah ada.');
            if (!rename($item, $dest)) throw new RuntimeException('Gagal mengubah nama.');
            flash('Nama berhasil diubah.');
        } elseif ($action === 'delete') {
            $rel = clean_rel((string)$_POST['item']);
            if ($rel === '') throw new RuntimeException('Folder utama tidak dapat dihapus.');
            $item = target_path($rel);
            if (is_dir($item)) {
                $contents = scandir($item);
                if (count($contents) > 2) throw new RuntimeException('Folder tidak kosong. Hapus isinya terlebih dahulu.');
                if (!rmdir($item)) throw new RuntimeException('Gagal menghapus folder.');
            } else {
                if (!unlink($item)) throw new RuntimeException('Gagal menghapus file.');
            }
            flash('Item berhasil dihapus.');
        }
    } catch (Throwable $ex) { flash($ex->getMessage(), 'error'); }
    redirect_to($cwd);
}

$items = [];
foreach (scandir($current) ?: [] as $name) {
    if ($name === '.' || $name === '..' || $name[0] === '.') continue;
    $full = $current . DIRECTORY_SEPARATOR . $name;
    // Symlink tidak ditampilkan untuk mencegah akses keluar dari direktori storage.
    if (is_link($full)) continue;
    $isDir = is_dir($full);
    $items[] = ['name'=>$name, 'dir'=>$isDir, 'size'=>$isDir ? 0 : filesize($full), 'time'=>filemtime($full), 'mime'=>$isDir ? 'Folder' : mime_type($full)];
}
usort($items, fn($a,$b) => ($b['dir'] <=> $a['dir']) ?: strcasecmp($a['name'],$b['name']));
$flash = $_SESSION['flash'] ?? null; unset($_SESSION['flash']);
$segments = $cwd === '' ? [] : explode('/', $cwd);
function fmt_size(int $n): string { $u=['B','KB','MB','GB']; $i=0; while($n>=1024 && $i<3){$n/=1024;$i++;} return number_format($n,$i?1:0,',','.').' '.$u[$i]; }
?><!doctype html><html lang="id"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1"><title><?=e(APP_NAME)?></title><link rel="stylesheet" href="assets/style.css"></head>
<body><header class="topbar"><div class="brand"><span class="brand-icon small">▦</span><strong><?=e(APP_NAME)?></strong></div><div class="top-actions"><?php if(logged()): ?><span class="user-chip">Administrator</span><a class="btn light" href="?logout=1">Keluar</a><?php else: ?><span class="user-chip">Akses Publik</span><a class="btn light" href="index.php?login=1">Login Admin</a><?php endif; ?></div></header>
<main class="container"><section class="welcome"><div><h1>File Sharing</h1><p><?php if(logged()): ?>Anda login sebagai administrator. Kelola dan bagikan file Anda.<?php else: ?>File Bantuan NPHD.<?php endif; ?></p></div><div class="stats"><span><b><?=count($items)?></b> item</span><span><b><?=count(array_filter($items,fn($x)=>$x['dir']))?></b> folder</span></div></section>
<?php if($flash): ?><div class="alert <?=e($flash[1])?>"><?=e($flash[0])?></div><?php endif; ?>
<section class="toolbar"><div class="breadcrumbs"><a href="/bantuan">🏠 Beranda</a><?php $acc=''; foreach($segments as $seg){$acc .= ($acc?'/':'').$seg; ?> <span>/</span> <a href="index.php?path=<?=rawurlencode($acc)?>"><?=e($seg)?></a><?php } ?></div>
<div class="tools"><input id="search" class="search" placeholder="Cari file atau folder..."><?php if(logged()): ?><button class="btn primary" onclick="document.getElementById('uploadInput').click()">＋ Upload file</button><button class="btn light" onclick="openModal('folderModal')">＋ Folder baru</button><?php endif; ?></div></section>
<?php if(logged()): ?><form id="uploadForm" method="post" enctype="multipart/form-data" class="hidden"><input type="hidden" name="csrf" value="<?=e(csrf())?>"><input type="hidden" name="do" value="upload"><input id="uploadInput" type="file" name="upload_files[]" multiple onchange="document.getElementById('uploadForm').submit()"></form><?php endif; ?>
<section class="file-panel"><div class="panel-head"><span>Nama</span><span>Ukuran</span><span>Jenis</span><span>Terakhir diubah</span><span><?=logged() ? "Aksi" : "Download"?></span></div>
<?php if($cwd!==''): ?><div class="file-row parent-row"><div class="file-name"><span class="file-icon">↩</span><a href="index.php?path=<?=rawurlencode(dirname($cwd)==='.'?'':dirname($cwd))?>">Kembali ke folder sebelumnya</a></div><span>—</span><span>—</span><span>—</span><span>—</span></div><?php endif; ?>
<?php foreach($items as $it): $rel=($cwd!==''?$cwd.'/':'').$it['name']; $ext=strtolower(pathinfo($it['name'],PATHINFO_EXTENSION)); $icon=$it['dir']?'📁':(in_array($ext,['jpg','jpeg','png','gif','webp'])?'🖼️':(in_array($ext,['pdf'])?'📕':(in_array($ext,['mp4','webm','mp3','wav'])?'🎬':'📄'))); ?>
<div class="file-row" data-name="<?=e(strtolower($it['name']))?>"><div class="file-name"><span class="file-icon"><?=$icon?></span><?php if($it['dir']): ?><a class="item-link" href="index.php?path=<?=rawurlencode($rel)?>"><?=e($it['name'])?></a><?php else: ?><button class="name-button item-link" onclick="previewFile('<?=e(rawurlencode($rel))?>','<?=e($it['name'])?>')"><?=e($it['name'])?></button><?php endif; ?></div><span><?= $it['dir']?'—':fmt_size((int)$it['size']) ?></span><span><?=e($it['mime'])?></span><span><?=date('d/m/Y H:i',(int)$it['time'])?></span><?php if(!$it['dir'] || logged()): ?><div class="actions"><?php if(!$it['dir']): ?><a title="Download" href="?action=download&file=<?=rawurlencode($rel)?>">⬇</a><?php endif; ?><?php if(logged()): ?><button title="Rename" onclick="renameItem('<?=e(rawurlencode($rel))?>','<?=e($it['name'])?>')">✎</button><button title="Hapus" class="danger-text" onclick="deleteItem('<?=e(rawurlencode($rel))?>','<?=e($it['name'])?>')">🗑</button><?php endif; ?></div><?php endif; ?></div>
<?php endforeach; ?>
<?php if(!$items): ?><div class="empty"><div>📂</div><strong>Folder ini masih kosong</strong><p>Upload file atau buat folder baru untuk memulai.</p></div><?php endif; ?>
</section><p class="footnote">Penyimpanan: <code>/storage</code> · Maksimum upload per file: <?=fmt_size(MAX_UPLOAD_BYTES)?></p></main>

<?php if(logged()): ?>
<div class="modal-backdrop" id="folderModal"><form class="modal" method="post"><button type="button" class="close" onclick="closeModal('folderModal')">×</button><h2>Buat folder baru</h2><input type="hidden" name="csrf" value="<?=e(csrf())?>"><input type="hidden" name="do" value="mkdir"><label>Nama folder</label><input name="name" required maxlength="120" placeholder="Contoh: Dokumen"><button class="btn primary full">Buat folder</button></form></div>
<div class="modal-backdrop" id="renameModal"><form class="modal" method="post"><button type="button" class="close" onclick="closeModal('renameModal')">×</button><h2>Ubah nama</h2><input type="hidden" name="csrf" value="<?=e(csrf())?>"><input type="hidden" name="do" value="rename"><input type="hidden" name="old" id="renameOld"><label>Nama baru</label><input name="new" id="renameNew" required maxlength="180"><button class="btn primary full">Simpan perubahan</button></form></div>
<form id="deleteForm" method="post" class="hidden"><input type="hidden" name="csrf" value="<?=e(csrf())?>"><input type="hidden" name="do" value="delete"><input type="hidden" name="item" id="deleteItem"></form>
<?php endif; ?>
<div class="modal-backdrop" id="previewModal"><div class="preview-box"><div class="preview-head"><strong id="previewTitle">Preview</strong><button class="close" onclick="closeModal('previewModal')">×</button></div><iframe id="previewFrame" title="Preview file"></iframe></div></div>
<script>
function openModal(id){document.getElementById(id).classList.add('show')}
function closeModal(id){document.getElementById(id).classList.remove('show');if(id==='previewModal')document.getElementById('previewFrame').src='about:blank'}
function renameItem(path,name){document.getElementById('renameOld').value=decodeURIComponent(path);document.getElementById('renameNew').value=name;openModal('renameModal')}
function deleteItem(path,name){if(confirm('Hapus "'+name+'"? Folder hanya dapat dihapus jika kosong.')){document.getElementById('deleteItem').value=decodeURIComponent(path);document.getElementById('deleteForm').submit()}}
function previewFile(path,name){document.getElementById('previewTitle').textContent=name;document.getElementById('previewFrame').src='?action=preview&file='+path;openModal('previewModal')}
document.getElementById('search').addEventListener('input',function(){let q=this.value.toLowerCase();document.querySelectorAll('.file-row[data-name]').forEach(r=>r.style.display=r.dataset.name.includes(q)?'grid':'none')});
document.querySelectorAll('.modal-backdrop').forEach(m=>m.addEventListener('click',e=>{if(e.target===m)closeModal(m.id)}));
</script></body></html>
