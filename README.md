<?php
// Database configuration
$host = '';
$dbname = '';
$username_db = '';
$password_db = '';

// Initialize variables
$uploadSuccess = false;
$uploadError = '';
$musicList = [];
$top30Songs = [];
$allSongs = [];
$searchResults = [];
$genres = [];
$activeTab = isset($_GET['tab']) ? $_GET['tab'] : 'home';
$songLink = '';
$currentGenre = isset($_GET['genre']) ? $_GET['genre'] : '';
$favoriteRadios = [];
$dailyUploadCount = 0;
$recommendedSongs = [];
$waitlistSuccess = false;
$waitlistError = '';
$waitlistName = '';

// Function to get correct thumbnail path
function getThumbnailPath($cover) {
    if ($cover === 'default_cover.jpg') {
        return 'default_cover.jpg';
    }
    
    if (file_exists('thumbnails/' . $cover)) {
        return 'thumbnails/' . $cover;
    } elseif (file_exists('uploads/' . $cover)) {
        return 'uploads/' . $cover;
    } else {
        return 'default_cover.jpg';
    }
}

// Function to get full URL for thumbnails
function getThumbnailUrl($cover) {
    $baseUrl = (isset($_SERVER['HTTPS']) && $_SERVER['HTTPS'] === 'on' ? "https" : "http") . "://$_SERVER[HTTP_HOST]";
    $thumbnailPath = getThumbnailPath($cover);
    
    if ($thumbnailPath === 'default_cover.jpg') {
        return $baseUrl . '/default_cover.jpg';
    }
    
    return $baseUrl . '/' . $thumbnailPath;
}

// Dynamic meta tags
$shareTitle = "AudioVA - Free Music Platform";
$shareDescription = "Malawi's premier free music streaming platform. Upload your music for free and connect with listeners worldwide.";
$shareImage = getThumbnailUrl('logo.png');
$shareUrl = (isset($_SERVER['HTTPS']) && $_SERVER['HTTPS'] === 'on' ? "https" : "http") . "://$_SERVER[HTTP_HOST]$_SERVER[REQUEST_URI]";
$shareSong = null;
$pageKeywords = "music, streaming, malawi, audio, songs, artists, free music, upload music, audio streaming, radio, malawian music";

// Check if we're viewing a specific song
if (isset($_GET['play']) && is_numeric($_GET['play'])) {
    try {
        $pdo_temp = new PDO("mysql:host=$host;dbname=$dbname", $username_db, $password_db);
        $pdo_temp->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        
        $song_id = intval($_GET['play']);
        $stmt = $pdo_temp->prepare("SELECT * FROM songs WHERE id = :song_id");
        $stmt->execute(['song_id' => $song_id]);
        $song = $stmt->fetch(PDO::FETCH_ASSOC);
        
        if ($song) {
            $shareTitle = htmlspecialchars($song['title'] . " by " . $song['artist'] . " | AudioVA");
            $shareDescription = "Listen to " . htmlspecialchars($song['title']) . " by " . htmlspecialchars($song['artist']) . " on AudioVA. Upload your music for free!";
            $shareImage = getThumbnailUrl($song['cover']);
            $shareSong = $song;
            $pageKeywords = htmlspecialchars($song['title'] . ", " . $song['artist'] . ", " . $song['genre'] . ", music, streaming, download");
        }
    } catch(Exception $e) {
        // Continue with default meta tags
    }
}

// Store form data in session
session_start();

// Create database connection
try {
    $pdo = new PDO("mysql:host=$host;dbname=$dbname", $username_db, $password_db);
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    
    // Create tables if they don't exist
    $pdo->exec("CREATE TABLE IF NOT EXISTS songs (
        id INT AUTO_INCREMENT PRIMARY KEY,
        title VARCHAR(255) NOT NULL,
        artist VARCHAR(255) NOT NULL,
        genre VARCHAR(100) NOT NULL DEFAULT 'Other',
        file VARCHAR(255) NOT NULL,
        cover VARCHAR(255) DEFAULT 'default_cover.jpg',
        streams INT DEFAULT 0,
        downloads INT DEFAULT 0,
        likes INT DEFAULT 0,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )");
    
    $pdo->exec("CREATE TABLE IF NOT EXISTS comments (
        id INT AUTO_INCREMENT PRIMARY KEY,
        song_id INT NOT NULL,
        user_name VARCHAR(100) NOT NULL,
        comment TEXT NOT NULL,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        FOREIGN KEY (song_id) REFERENCES songs(id) ON DELETE CASCADE
    )");
    
    // Create radio favorites table
    $pdo->exec("CREATE TABLE IF NOT EXISTS radio_favorites (
        id INT AUTO_INCREMENT PRIMARY KEY,
        radio_id VARCHAR(100) NOT NULL,
        radio_name VARCHAR(255) NOT NULL,
        radio_stream VARCHAR(500) NOT NULL,
        radio_logo VARCHAR(500) DEFAULT 'radio_default.png',
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )");
    
    // Create waitlist table for earn feature
    $pdo->exec("CREATE TABLE IF NOT EXISTS earn_waitlist (
        id INT AUTO_INCREMENT PRIMARY KEY,
        name VARCHAR(255) NOT NULL,
        email VARCHAR(255),
        phone VARCHAR(50),
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )");

    // Get waitlist count
    $waitlistCount = 0;
    try {
        $stmt = $pdo->query("SELECT COUNT(*) as count FROM earn_waitlist");
        $result = $stmt->fetch(PDO::FETCH_ASSOC);
        $waitlistCount = $result['count'] ?? 0;
    } catch(Exception $e) {
        error_log("Waitlist count error: " . $e->getMessage());
    }
    
    // Fetch data with limits for performance
    $stmt = $pdo->query("SELECT * FROM songs ORDER BY streams DESC LIMIT 30");
    $top30Songs = $stmt->fetchAll(PDO::FETCH_ASSOC);
    
    $stmt = $pdo->query("SELECT * FROM songs ORDER BY id DESC LIMIT 20");
    $musicList = $stmt->fetchAll(PDO::FETCH_ASSOC);
    
    // Get all genres with song counts
    $stmt = $pdo->query("SELECT genre, COUNT(*) as song_count FROM songs GROUP BY genre ORDER BY song_count DESC");
    $genres = $stmt->fetchAll(PDO::FETCH_ASSOC);
    
    $stmt = $pdo->query("SELECT * FROM songs ORDER BY id DESC");
    $allSongs = $stmt->fetchAll(PDO::FETCH_ASSOC);
    
    // Get favorite radios
    $stmt = $pdo->query("SELECT * FROM radio_favorites ORDER BY created_at DESC");
    $favoriteRadios = $stmt->fetchAll(PDO::FETCH_ASSOC);
    
    // Get trending songs (songs with most streams in last 7 days)
    $stmt = $pdo->query("SELECT * FROM songs WHERE created_at >= DATE_SUB(NOW(), INTERVAL 7 DAY) ORDER BY streams DESC LIMIT 10");
    $trendingSongs = $stmt->fetchAll(PDO::FETCH_ASSOC);
    
    // Get daily upload count (songs uploaded in last 24 hours)
    $stmt = $pdo->query("SELECT COUNT(*) as daily_count FROM songs WHERE created_at >= DATE_SUB(NOW(), INTERVAL 24 HOUR)");
    $dailyUploadResult = $stmt->fetch(PDO::FETCH_ASSOC);
    $dailyUploadCount = $dailyUploadResult['daily_count'] ?? 0;
    
    // Get songs by genre if genre filter is set
    if ($currentGenre) {
        $stmt = $pdo->prepare("SELECT * FROM songs WHERE genre = ? ORDER BY id DESC");
        $stmt->execute([$currentGenre]);
        $genreSongs = $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
    
    // Get recommended songs
    if (isset($_GET['play']) && is_numeric($_GET['play'])) {
        $currentSongId = intval($_GET['play']);
        $stmt = $pdo->prepare("SELECT * FROM songs WHERE id = ?");
        $stmt->execute([$currentSongId]);
        $currentSong = $stmt->fetch(PDO::FETCH_ASSOC);
        
        if ($currentSong) {
            $stmt = $pdo->prepare("SELECT * FROM songs WHERE genre = ? AND id != ? ORDER BY RAND() LIMIT 5");
            $stmt->execute([$currentSong['genre'], $currentSongId]);
            $recommendedSongs = $stmt->fetchAll(PDO::FETCH_ASSOC);
            
            if (count($recommendedSongs) < 5) {
                $stmt = $pdo->prepare("SELECT * FROM songs WHERE id != ? ORDER BY RAND() LIMIT ?");
                $remaining = 5 - count($recommendedSongs);
                $stmt->bindValue(1, $currentSongId, PDO::PARAM_INT);
                $stmt->bindValue(2, $remaining, PDO::PARAM_INT);
                $stmt->execute();
                $additionalSongs = $stmt->fetchAll(PDO::FETCH_ASSOC);
                $recommendedSongs = array_merge($recommendedSongs, $additionalSongs);
            }
        }
    }
    
} catch(PDOException $e) {
    $uploadError = "Database connection failed: " . $e->getMessage();
}

// Handle search
if (isset($_GET['q']) && !empty($_GET['q'])) {
    $searchTerm = $_GET['q'];
    try {
        $stmt = $pdo->prepare("SELECT * FROM songs WHERE title LIKE ? OR artist LIKE ? OR genre LIKE ? ORDER BY id DESC LIMIT 50");
        $stmt->execute(["%$searchTerm%", "%$searchTerm%", "%$searchTerm%"]);
        $searchResults = $stmt->fetchAll(PDO::FETCH_ASSOC);
    } catch(Exception $e) {
        $searchResults = [];
    }
}

// Handle song play
if (isset($_GET['play']) && is_numeric($_GET['play'])) {
    $song_id = intval($_GET['play']);
    try {
        $stmt = $pdo->prepare("SELECT * FROM songs WHERE id = ?");
        $stmt->execute([$song_id]);
        $currentSong = $stmt->fetch(PDO::FETCH_ASSOC);
        if ($currentSong) {
            $stmt = $pdo->prepare("UPDATE songs SET streams = streams + 1 WHERE id = ?");
            $stmt->execute([$song_id]);
            $popupSong = $currentSong;
            
            $stmt = $pdo->prepare("SELECT * FROM comments WHERE song_id = ? ORDER BY created_at DESC LIMIT 50");
            $stmt->execute([$song_id]);
            $comments = $stmt->fetchAll(PDO::FETCH_ASSOC);
        }
    } catch(Exception $e) {
        $currentSong = null;
        $comments = [];
    }
}

// Handle download
if (isset($_GET['download']) && is_numeric($_GET['download'])) {
    $song_id = intval($_GET['download']);
    try {
        $stmt = $pdo->prepare("SELECT * FROM songs WHERE id = ?");
        $stmt->execute([$song_id]);
        $song = $stmt->fetch(PDO::FETCH_ASSOC);
        if ($song) {
            $stmt = $pdo->prepare("UPDATE songs SET downloads = downloads + 1 WHERE id = ?");
            $stmt->execute([$song_id]);
            
            $file_path = 'uploads/' . $song['file'];
            if (file_exists($file_path)) {
                header('Content-Type: audio/mpeg');
                header('Content-Disposition: attachment; filename="' . $song['title'] . ' - ' . $song['artist'] . '.mp3"');
                header('Content-Length: ' . filesize($file_path));
                readfile($file_path);
                exit;
            }
        }
    } catch(Exception $e) {
        // Error handling
    }
}

// Handle like
if (isset($_GET['like']) && is_numeric($_GET['like'])) {
    $song_id = intval($_GET['like']);
    try {
        $stmt = $pdo->prepare("UPDATE songs SET likes = likes + 1 WHERE id = ?");
        $stmt->execute([$song_id]);
        $stmt = $pdo->prepare("SELECT likes FROM songs WHERE id = ?");
        $stmt->execute([$song_id]);
        $likes = $stmt->fetchColumn();
        echo $likes;
        exit;
    } catch(Exception $e) {
        echo "0";
        exit;
    }
}

// Handle radio favorite
if (isset($_GET['favorite_radio']) && !empty($_GET['favorite_radio'])) {
    $radio_id = $_GET['favorite_radio'];
    $radio_name = $_GET['radio_name'] ?? '';
    $radio_stream = $_GET['radio_stream'] ?? '';
    $radio_logo = $_GET['radio_logo'] ?? 'radio_default.png';
    
    try {
        $stmt = $pdo->prepare("SELECT id FROM radio_favorites WHERE radio_id = ?");
        $stmt->execute([$radio_id]);
        $existing = $stmt->fetch();
        
        if (!$existing) {
            $stmt = $pdo->prepare("INSERT INTO radio_favorites (radio_id, radio_name, radio_stream, radio_logo) VALUES (?, ?, ?, ?)");
            $stmt->execute([$radio_id, $radio_name, $radio_stream, $radio_logo]);
        }
        
        echo "success";
        exit;
    } catch(Exception $e) {
        echo "error";
        exit;
    }
}

// Handle radio unfavorite
if (isset($_GET['unfavorite_radio']) && !empty($_GET['unfavorite_radio'])) {
    $radio_id = $_GET['unfavorite_radio'];
    
    try {
        $stmt = $pdo->prepare("DELETE FROM radio_favorites WHERE radio_id = ?");
        $stmt->execute([$radio_id]);
        echo "success";
        exit;
    } catch(Exception $e) {
        echo "error";
        exit;
    }
}

// Handle comment submission
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['submit_comment'])) {
    $song_id = intval($_POST['song_id']);
    $user_name = trim($_POST['user_name'] ?? '');
    $comment = trim($_POST['comment'] ?? '');
    
    if (!empty($user_name) && !empty($comment) && $song_id > 0) {
        try {
            $stmt = $pdo->prepare("INSERT INTO comments (song_id, user_name, comment) VALUES (?, ?, ?)");
            $stmt->execute([$song_id, $user_name, $comment]);
            
            header("Location: ?play=" . $song_id . "&tab=" . $activeTab);
            exit;
        } catch(Exception $e) {
            $commentError = "Failed to post comment: " . $e->getMessage();
        }
    }
}

// Handle waitlist submission
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['join_waitlist'])) {
    $name = trim($_POST['waitlist_name'] ?? '');
    $email = trim($_POST['waitlist_email'] ?? '');
    $phone = trim($_POST['waitlist_phone'] ?? '');
    
    if (empty($name)) {
        $waitlistError = "Name is required to join the waitlist.";
    } else {
        try {
            $stmt = $pdo->prepare("INSERT INTO earn_waitlist (name, email, phone) VALUES (?, ?, ?)");
            $stmt->execute([$name, $email, $phone]);
            
            $waitlistSuccess = true;
            $waitlistName = $name;
            
            $_POST['waitlist_name'] = '';
            $_POST['waitlist_email'] = '';
            $_POST['waitlist_phone'] = '';
            
        } catch (Exception $e) {
            $waitlistError = "Failed to join waitlist: " . $e->getMessage();
        }
    }
}

// Handle upload form submission
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['upload_info'])) {
    $title = trim($_POST['title'] ?? '');
    $artist = trim($_POST['artist'] ?? '');
    $genre = trim($_POST['genre'] ?? 'Other');
    $phone = trim($_POST['phone'] ?? '');
    $location = trim($_POST['location'] ?? '');
    $audioFile = $_FILES['audio_file'] ?? null;
    $thumbnailFile = $_FILES['thumbnail'] ?? null;
    
    if (empty($title)) {
        $uploadError = "Song title is required.";
    } elseif (empty($artist)) {
        $uploadError = "Artist name is required.";
    } elseif (empty($genre)) {
        $uploadError = "Genre is required.";
    } elseif (empty($phone)) {
        $uploadError = "Phone number is required.";
    } elseif (empty($location)) {
        $uploadError = "Location is required.";
    } elseif (!$audioFile || $audioFile['error'] !== UPLOAD_ERR_OK) {
        $uploadError = "Please select a valid audio file.";
    } elseif ($audioFile['size'] > 7 * 1024 * 1024) {
        $uploadError = "Audio file must be less than 7MB.";
    } elseif ($thumbnailFile && $thumbnailFile['error'] !== UPLOAD_ERR_OK && $thumbnailFile['error'] !== UPLOAD_ERR_NO_FILE) {
        $uploadError = "Invalid thumbnail file.";
    } elseif ($thumbnailFile && $thumbnailFile['size'] > 2 * 1024 * 1024) {
        $uploadError = "Thumbnail must be less than 2MB.";
    } else {
        try {
            $audioExtension = pathinfo($audioFile['name'], PATHINFO_EXTENSION);
            $audioFilename = uniqid() . '.' . $audioExtension;
            
            $thumbnailFilename = 'default_cover.jpg';
            if ($thumbnailFile && $thumbnailFile['error'] === UPLOAD_ERR_OK) {
                $thumbnailExtension = pathinfo($thumbnailFile['name'], PATHINFO_EXTENSION);
                $thumbnailFilename = uniqid() . '.' . $thumbnailExtension;
            }
            
            $_SESSION['artist_contact'] = [
                'phone' => $phone,
                'location' => $location
            ];
            
            $stmt = $pdo->prepare("INSERT INTO songs (title, artist, genre, file, cover) VALUES (?, ?, ?, ?, ?)");
            $stmt->execute([$title, $artist, $genre, $audioFilename, $thumbnailFilename]);
            $songId = $pdo->lastInsertId();
            
            if (!file_exists('uploads')) mkdir('uploads', 0777, true);
            if (!file_exists('thumbnails')) mkdir('thumbnails', 0777, true);
            
            if (!move_uploaded_file($audioFile['tmp_name'], 'uploads/' . $audioFilename)) {
                throw new Exception("Failed to upload audio file");
            }
            
            if ($thumbnailFilename !== 'default_cover.jpg') {
                if (!move_uploaded_file($thumbnailFile['tmp_name'], 'thumbnails/' . $thumbnailFilename)) {
                    throw new Exception("Failed to upload thumbnail");
                }
            }
            
            $uploadSuccess = true;
            $songLink = (isset($_SERVER['HTTPS']) && $_SERVER['HTTPS'] === 'on' ? "https" : "http") . "://$_SERVER[HTTP_HOST]$_SERVER[SCRIPT_NAME]?play=" . $songId . "&tab=home";
            
        } catch (Exception $e) {
            $uploadError = "Upload failed: " . $e->getMessage();
        }
    }
}

// Malawian Radio Stations Data
$malawianRadios = [
    [
        'id' => 'mbc_radio_1',
        'name' => 'MBC Radio 1',
        'stream' => 'https://stream.zeno.fm/1yrtv8b8n2zuv',
        'logo' => 'https://audiova.unaux.com/mbc.jpg',
        'frequency' => 'FM 101.7',
        'location' => 'Blantyre'
    ],
    [
        'id' => 'mbc_radio_2',
        'name' => 'MBC Radio 2',
        'stream' => 'https://stream.zeno.fm/f4w0mbqdn2zuv',
        'logo' => 'https://audiova.unaux.com/mbc2.jpg',
        'frequency' => 'FM 93.1',
        'location' => 'Lilongwe'
    ],
    [
        'id' => 'zodiak_radio',
        'name' => 'Zodiak Radio',
        'stream' => 'https://ice31.securenetsystems.net/0079',
        'logo' => 'https://audiova.unaux.com/zodiak.png',
        'frequency' => 'FM 97.5',
        'location' => 'Nationwide'
    ],
    [
        'id' => 'times_radio',
        'name' => 'Times Radio',
        'stream' => 'https://ice31.securenetsystems.net/TIMESFM',
        'logo' => 'https://audiova.unaux.com/times.jpg',
        'frequency' => 'FM 88.8',
        'location' => 'Blantyre'
    ],
    [
        'id' => 'capital_radio',
        'name' => 'Capital Radio',
        'stream' => 'https://stream.zeno.fm/cca1x9s4n2zuv',
        'logo' => 'https://i.ibb.co/0Q8L8w2/mbc1.png',
        'frequency' => 'FM 100.5',
        'location' => 'Lilongwe'
    ],
    [
        'id' => 'joy_radio',
        'name' => 'Joy Radio',
        'stream' => 'https://stream.zeno.fm/2w0x8b8n2zuv',
        'logo' => 'https://i.ibb.co/0Q8L8w2/mbc1.png',
        'frequency' => 'FM 89.1',
        'location' => 'Blantyre'
    ],
    [
        'id' => 'power_101',
        'name' => 'Power 101 FM',
        'stream' => 'https://stream.zeno.fm/3yrtv8b8n2zuv',
        'logo' => 'https://i.ibb.co/0Q8L8w2/mbc1.png',
        'frequency' => 'FM 101.1',
        'location' => 'Lilongwe'
    ],
    [
        'id' => 'angaliba_radio',
        'name' => 'Angaliba Radio',
        'stream' => 'https://stream.zeno.fm/5w0x8b8n2zuv',
        'logo' => 'https://i.ibb.co/0Q8L8w2/mbc1.png',
        'frequency' => 'FM 90.3',
        'location' => 'Mzuzu'
    ],
    [
        'id' => 'mij_radio',
        'name' => 'MIJ Radio',
        'stream' => 'https://stream.zeno.fm/6yrtv8b8n2zuv',
        'logo' => 'https://i.ibb.co/0Q8L8w2/mbc1.png',
        'frequency' => 'FM 90.5',
        'location' => 'Lilongwe'
    ],
    [
        'id' => 'radio_maria',
        'name' => 'Radio Maria',
        'stream' => 'https://dreamsiteradiocp2.com/proxy/rmmalawi2?mp=/stream',
        'logo' => 'https://audiova.unaux.com/radiomaria.png',
        'frequency' => 'FM 101.3',
        'location' => 'Blantyre'
    ]
];
?>
<!DOCTYPE html>
<html lang="en">
<head>
    <!-- All the meta tags, styles, and scripts from original file -->
    <!-- ... (keep all the head content from original file) ... -->
</head>
<body>
    <!-- iOS Dynamic Island APK Download -->
    <div class="ios-dynamic-island" id="iosDynamicIsland">
        <!-- ... (keep iOS dynamic island content) ... -->
    </div>

    <!-- Header -->
    <header class="header">
        <div class="container">
            <div class="header-content">
                <a href="?tab=home" class="logo">
                    <i class="fas fa-music"></i>
                    AudioVA
                </a>
                
                <!-- Navigation -->
                <nav class="nav-tabs">
                    <button class="nav-tab <?= $activeTab === 'home' ? 'active' : '' ?>" onclick="switchTab('home')">
                        <i class="fas fa-home"></i>
                        <span>Home</span>
                        <?php if ($dailyUploadCount > 0): ?>
                            <span class="daily-counter"><?= $dailyUploadCount ?></span>
                        <?php endif; ?>
                    </button>
                    <button class="nav-tab <?= $activeTab === 'browse' ? 'active' : '' ?>" onclick="switchTab('browse')">
                        <i class="fas fa-compass"></i>
                        <span>Browse</span>
                    </button>
                    <button class="nav-tab <?= $activeTab === 'charts' ? 'active' : '' ?>" onclick="switchTab('charts')">
                        <i class="fas fa-chart-line"></i>
                        <span>Top 30</span>
                    </button>
                    <button class="nav-tab <?= $activeTab === 'radio' ? 'active' : '' ?>" onclick="switchTab('radio')">
                        <i class="fas fa-radio"></i>
                        <span>Radio</span>
                    </button>
                    <button class="nav-tab <?= $activeTab === 'upload' ? 'active' : '' ?>" onclick="switchTab('upload')">
                        <i class="fas fa-upload"></i>
                        <span>Upload</span>
                    </button>
                    <button class="nav-tab <?= $activeTab === 'earn' ? 'active' : '' ?>" onclick="switchTab('earn')">
                        <i class="fas fa-money-bill-wave"></i>
                        <span>Earn</span>
                    </button>
                </nav>
            </div>
        </div>
    </header>

    <div class="container">
        <!-- Search Box -->
        <form method="GET" class="search-box">
            <input type="text" name="q" placeholder="Search songs, artists, or genres..." value="<?= htmlspecialchars($_GET['q'] ?? '') ?>">
            <button type="submit"><i class="fas fa-search"></i> Search</button>
        </form>

        <!-- Search Results -->
        <?php if (isset($_GET['q']) && !empty($_GET['q'])): ?>
            <div class="section-header">
                <h2 class="section-title">Search Results for "<?= htmlspecialchars($_GET['q']) ?>"</h2>
            </div>
            <?php if (count($searchResults) > 0): ?>
                <div class="search-results-list">
                    <?php foreach ($searchResults as $song): ?>
                        <div class="search-result-item" onclick="window.location.href='player.php?id=<?= $song['id'] ?>&ref=search'">
                            <?php if ($song['cover'] === 'default_cover.jpg'): ?>
                                <div class="default-thumbnail" style="width: 60px; height: 60px; font-size: 16px;">
                                    <i class="fas fa-music"></i>
                                </div>
                            <?php else: ?>
                                <img src="<?= getThumbnailUrl($song['cover']) ?>" alt="Cover" class="search-result-cover">
                            <?php endif; ?>
                            
                            <div class="search-result-info">
                                <div class="search-result-title"><?= htmlspecialchars($song['title']) ?></div>
                                <div class="search-result-artist"><?= htmlspecialchars($song['artist']) ?></div>
                                <div class="search-result-uploaded">
                                    <i class="far fa-calendar-alt"></i> <?= date('M d, Y', strtotime($song['created_at'])) ?>
                                </div>
                                <div class="search-result-genre"><?= htmlspecialchars($song['genre']) ?></div>
                            </div>
                            
                            <div class="search-result-stats">
                                <div class="search-result-stat">
                                    <i class="fas fa-play"></i>
                                    <span><?= $song['streams'] ?></span>
                                </div>
                                <div class="search-result-stat">
                                    <i class="fas fa-download"></i>
                                    <span><?= $song['downloads'] ?></span>
                                </div>
                            </div>
                        </div>
                    <?php endforeach; ?>
                </div>
            <?php else: ?>
                <p>No songs found for "<?= htmlspecialchars($_GET['q']) ?>"</p>
            <?php endif; ?>
        <?php endif; ?>

        <!-- Tab Content -->
        <?php
        // Include the appropriate tab file
        $tabFile = 'tabs/' . $activeTab . '.php';
        if (file_exists($tabFile)) {
            include($tabFile);
        } else {
            include('tabs/home.php');
        }
        ?>
    </div>

    <!-- Footer -->
    <footer>
        <!-- ... (keep footer content from original file) ... -->
    </footer>

    <script>
        // JavaScript functions from original file
        // ... (keep all JavaScript functions from original file) ...
    </script>
</body>
</html>
