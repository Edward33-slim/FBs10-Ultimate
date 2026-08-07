package com.fbs10.lite

import android.annotation.SuppressLint
import android.content.Context
import android.content.SharedPreferences
import android.graphics.Color
import android.net.ConnectivityManager
import android.os.Build
import android.os.Bundle
import android.view.LayoutInflater
import android.view.View
import android.webkit.WebChromeClient
import android.webkit.WebResourceRequest
import android.webkit.WebResourceResponse
import android.webkit.WebSettings
import android.webkit.WebView
import android.webkit.WebViewClient
import android.widget.ImageView
import android.widget.TextView
import android.widget.Toast
import androidx.appcompat.app.AlertDialog
import androidx.appcompat.app.AppCompatActivity
import androidx.swiperefreshlayout.widget.SwipeRefreshLayout
import com.google.android.material.bottomsheet.BottomSheetDialog
import com.google.android.material.tabs.TabLayout
import java.io.ByteArrayInputStream

class MainActivity : AppCompatActivity() {

    private lateinit var webView: WebView
    private lateinit var swipeRefresh: SwipeRefreshLayout
    private lateinit var prefs: SharedPreferences

    private var isAdsHidden = true
    private var isMediaHidden = true
    private var isSuggestionsHidden = true
    private var currentTheme = "light"

    private val favorites = mutableListOf<String>()

    private lateinit var btnHideAds: ImageView
    private lateinit var btnHideMedia: ImageView
    private lateinit var btnHideSuggestions: ImageView
    private lateinit var btnTheme: ImageView
    private lateinit var btnAddFavorite: ImageView
    private lateinit var btnShowFavorites: ImageView
    private lateinit var badgeNotifications: TextView
    private lateinit var badgeMessages: TextView

    @SuppressLint("SetJavaScriptEnabled")
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        prefs = getSharedPreferences("fbs10_prefs", Context.MODE_PRIVATE)
        loadPreferences()
        loadFavorites()

        webView = findViewById(R.id.webView)
        swipeRefresh = findViewById(R.id.swipeRefresh)

        btnHideAds = findViewById(R.id.btnHideAds)
        btnHideMedia = findViewById(R.id.btnHideMedia)
        btnHideSuggestions = findViewById(R.id.btnHideSuggestions)
        btnTheme = findViewById(R.id.btnTheme)
        btnAddFavorite = findViewById(R.id.btnAddFavorite)
        btnShowFavorites = findViewById(R.id.btnShowFavorites)
        badgeNotifications = findViewById(R.id.badgeNotifications)
        badgeMessages = findViewById(R.id.badgeMessages)

        setupWebView()
        setupControls()
        updateButtonStates()

        webView.loadUrl(getBestFacebookUrl())
    }

    private fun loadPreferences() {
        isAdsHidden = prefs.getBoolean("hide_ads", true)
        isMediaHidden = prefs.getBoolean("hide_media", true)
        isSuggestionsHidden = prefs.getBoolean("hide_suggestions", true)
        currentTheme = prefs.getString("theme", "light") ?: "light"
    }

    private fun savePreferences() {
        prefs.edit().apply {
            putBoolean("hide_ads", isAdsHidden)
            putBoolean("hide_media", isMediaHidden)
            putBoolean("hide_suggestions", isSuggestionsHidden)
            putString("theme", currentTheme)
            apply()
        }
    }

    private fun loadFavorites() {
        favorites.clear()
        val list = prefs.getString("favorites_list", "")
        if (!list.isNullOrEmpty()) {
            favorites.addAll(list.split("|||").filter { it.isNotEmpty() })
        }
    }

    private fun saveFavorites() {
        prefs.edit().putString("favorites_list", favorites.joinToString("|||")).apply()
    }

    private fun updateButtonStates() {
        btnHideAds.setColorFilter(if (isAdsHidden) Color.GRAY else Color.WHITE)
        btnHideMedia.setColorFilter(if (isMediaHidden) Color.GRAY else Color.WHITE)
        btnHideSuggestions.setColorFilter(if (isSuggestionsHidden) Color.GRAY else Color.WHITE)
        btnTheme.setColorFilter(when (currentTheme) {
            "dim" -> Color.DKGRAY
            "amoled" -> Color.BLACK
            else -> Color.WHITE
        })
    }

    @SuppressLint("SetJavaScriptEnabled")
    private fun setupWebView() {
        val s = webView.settings
        s.javaScriptEnabled = true
        s.domStorageEnabled = true
        s.databaseEnabled = true
        s.cacheMode = WebSettings.LOAD_CACHE_ELSE_NETWORK
        s.mixedContentMode = WebSettings.MIXED_CONTENT_ALWAYS_ALLOW
        s.useWideViewPort = true
        s.loadWithOverviewMode = true
        s.userAgentString = "Mozilla/5.0 (Linux; Android 10; K) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Mobile Safari/537.36"
        if (Build.VERSION.SDK_INT >= 24) WebView.enableSlowWholeDocumentDraw()
        webView.setLayerType(View.LAYER_TYPE_HARDWARE, null)

        webView.webViewClient = object : WebViewClient() {
            override fun shouldInterceptRequest(view: WebView, request: WebResourceRequest): WebResourceResponse? {
                val url = request.url.toString()
                if (url.contains("doubleclick.net") || url.contains("googlesyndication")) {
                    return WebResourceResponse("text/plain", "utf-8", ByteArrayInputStream("".toByteArray()))
                }
                if (isMediaHidden) {
                    if (url.matches(Regex(".*\\.(jpg|jpeg|png|webp|mp4|m4v)(\\?.*)?$", RegexOption.IGNORE_CASE))) {
                        if (url.contains("emoji") || url.contains("reaction") || url.contains("sticker")) {
                            return super.shouldInterceptRequest(view, request)
                        }
                        return WebResourceResponse("text/plain", "utf-8", ByteArrayInputStream("".toByteArray()))
                    }
                }
                return super.shouldInterceptRequest(view, request)
            }

            override fun onPageFinished(view: WebView?, url: String?) {
                super.onPageFinished(view, url)
                swipeRefresh.isRefreshing = false
                injectAll()
                fetchBadges()
                expandComments()
            }

            override fun onReceivedError(view: WebView?, request: WebResourceRequest?, error: WebResourceError?) {
                super.onReceivedError(view, request, error)
                swipeRefresh.isRefreshing = false
            }
        }

        webView.webChromeClient = WebChromeClient()
        swipeRefresh.setOnRefreshListener { webView.reload() }
    }

    private fun setupControls() {
        btnHideAds.setOnClickListener {
            isAdsHidden = !isAdsHidden
            savePreferences()
            updateButtonStates()
            injectAds()
            Toast.makeText(this, if (isAdsHidden) "تم حجب الإعلانات" else "إظهار الإعلانات", Toast.LENGTH_SHORT).show()
        }

        btnHideMedia.setOnClickListener {
            isMediaHidden = !isMediaHidden
            savePreferences()
            updateButtonStates()
            injectMedia()
            webView.reload()
            Toast.makeText(this, if (isMediaHidden) "تم حجب الصور والفيديو" else "إظهار الصور والفيديو", Toast.LENGTH_SHORT).show()
        }

        btnHideSuggestions.setOnClickListener {
            isSuggestionsHidden = !isSuggestionsHidden
            savePreferences()
            updateButtonStates()
            injectSuggestions()
            Toast.makeText(this, if (isSuggestionsHidden) "تم حجب الاقتراحات" else "إظهار الاقتراحات", Toast.LENGTH_SHORT).show()
        }

        btnTheme.setOnClickListener {
            currentTheme = when (currentTheme) {
                "light" -> "dim"
                "dim" -> "amoled"
                else -> "light"
            }
            savePreferences()
            updateButtonStates()
            injectTheme()
            Toast.makeText(this, "الثيم: $currentTheme", Toast.LENGTH_SHORT).show()
        }

        btnAddFavorite.setOnClickListener {
            val url = webView.url ?: return@setOnClickListener
            if (url.contains("m.facebook.com") || url.contains("mbasic.facebook.com")) {
                if (!favorites.contains(url)) {
                    favorites.add(url)
                    saveFavorites()
                    Toast.makeText(this, "تمت إضافة المنشور إلى المفضلة", Toast.LENGTH_SHORT).show()
                } else {
                    Toast.makeText(this, "موجود بالفعل في المفضلة", Toast.LENGTH_SHORT).show()
                }
            } else {
                Toast.makeText(this, "يرجى فتح منشور أولاً", Toast.LENGTH_SHORT).show()
            }
        }

        btnShowFavorites.setOnClickListener {
            if (favorites.isEmpty()) {
                Toast.makeText(this, "لا توجد مفضلات", Toast.LENGTH_SHORT).show()
                return@setOnClickListener
            }
            val items = favorites.toTypedArray()
            AlertDialog.Builder(this)
                .setTitle("المفضلات")
                .setItems(items) { _, which ->
                    webView.loadUrl(items[which])
                }
                .setNegativeButton("إغلاق", null)
                .show()
        }

        findViewById<View>(R.id.btnNotifications).setOnClickListener {
            webView.loadUrl("https://m.facebook.com/notifications/")
        }

        findViewById<View>(R.id.btnMessenger).setOnClickListener {
            showMessengerDialog()
        }
    }

    @SuppressLint("SetJavaScriptEnabled", "InflateParams")
    private fun showMessengerDialog() {
        val dialog = BottomSheetDialog(this)
        val view = LayoutInflater.from(this).inflate(R.layout.dialog_messenger, null)
        dialog.setContentView(view)

        val tabLayout = view.findViewById<TabLayout>(R.id.tabLayoutMessages)
        val innerWebView = view.findViewById<WebView>(R.id.innerWebView)

        val s = innerWebView.settings
        s.javaScriptEnabled = true
        s.domStorageEnabled = true
        s.databaseEnabled = true
        s.cacheMode = WebSettings.LOAD_CACHE_ELSE_NETWORK
        s.mixedContentMode = WebSettings.MIXED_CONTENT_ALWAYS_ALLOW
        s.useWideViewPort = true
        s.loadWithOverviewMode = true
        s.userAgentString = "Mozilla/5.0 (Linux; Android 10; K) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Mobile Safari/537.36"
        innerWebView.setLayerType(View.LAYER_TYPE_HARDWARE, null)

        innerWebView.webViewClient = object : WebViewClient() {
            override fun shouldOverrideUrlLoading(view: WebView?, request: WebResourceRequest?): Boolean {
                return false
            }
        }
        innerWebView.webChromeClient = WebChromeClient()

        tabLayout.addTab(tabLayout.newTab().setText("الأساسية"))
        tabLayout.addTab(tabLayout.newTab().setText("الطلبات"))
        tabLayout.addTab(tabLayout.newTab().setText("غير مهم"))

        innerWebView.loadUrl("https://m.facebook.com/messages/")

        tabLayout.addOnTabSelectedListener(object : TabLayout.OnTabSelectedListener {
            override fun onTabSelected(tab: TabLayout.Tab?) {
                when (tab?.position) {
                    0 -> innerWebView.loadUrl("https://m.facebook.com/messages/")
                    1 -> innerWebView.loadUrl("https://m.facebook.com/messages/requests/")
                    2 -> innerWebView.loadUrl("https://m.facebook.com/messages/others/")
                }
            }
            override fun onTabUnselected(tab: TabLayout.Tab?) {}
            override fun onTabReselected(tab: TabLayout.Tab?) {}
        })

        view.findViewById<ImageView>(R.id.btnMessengerSettings).setOnClickListener {
            innerWebView.loadUrl("https://m.facebook.com/messages/settings/")
        }

        dialog.show()
    }

    private fun injectAll() {
        injectMedia()
        injectAds()
        injectSuggestions()
        injectTheme()
    }

    private fun injectMedia() {
        val js = """(function() {
                var hide = $isMediaHidden;
                if (!hide) return;
                var elements = document.querySelectorAll('img, video');
                for (var i = 0; i < elements.length; i++) {
                    var el = elements[i];
                    var src = el.src || '';
                    if (src.includes('emoji') || src.includes('reaction') || src.includes('sticker')) continue;
                    if (el.clientWidth < 40 && el.clientHeight < 40) continue;
                    el.style.display = 'none';
                }
            })();""".trimIndent()
        webView.evaluateJavascript(js, null)
    }

    private fun injectAds() {
        if (!isAdsHidden) {
            val jsShow = """(function() {
                    document.querySelectorAll('[data-fbs10-ads]').forEach(function(el) {
                        el.style.display = '';
                        el.removeAttribute('data-fbs10-ads');
                    });
                })();""".trimIndent()
            webView.evaluateJavascript(jsShow, null)
            return
        }
        val js = """(function() {
                var keywords = ['ممول', 'Sponsored', 'إعلان', 'Promoted', 'رصيد إعلانات', 'عزز نتائجك', 'TikTok'];
                var items = document.querySelectorAll('div, span, article, section');
                for (var i = 0; i < items.length; i++) {
                    var el = items[i];
                    var text = el.innerText || '';
                    var found = false;
                    for (var k = 0; k < keywords.length; k++) {
                        if (text.includes(keywords[k])) { found = true; break; }
                    }
                    if (found) {
                        var parent = el.closest('[role="article"]') || el.parentElement;
                        if (parent) {
                            parent.style.display = 'none';
                            parent.setAttribute('data-fbs10-ads', 'true');
                        }
                    }
                }
            })();""".trimIndent()
        webView.evaluateJavascript(js, null)
    }

    private fun injectSuggestions() {
        if (!isSuggestionsHidden) {
            val jsShow = """(function() {
                    document.querySelectorAll('[data-fbs10-sug]').forEach(function(el) {
                        el.style.display = '';
                        el.removeAttribute('data-fbs10-sug');
                    });
                })();""".trimIndent()
            webView.evaluateJavascript(jsShow, null)
            return
        }
        val js = """(function() {
                var targets = ['قد تعجبك', 'أشخاص قد تعرفهم', 'ريلز', 'قصص', 'صفحات قد تعجبك', 'اقتراحات'];
                var items = document.querySelectorAll('div, section, article');
                for (var i = 0; i < items.length; i++) {
                    var el = items[i];
                    var text = el.innerText || '';
                    var found = false;
                    for (var t = 0; t < targets.length; t++) {
                        if (text.includes(targets[t])) { found = true; break; }
                    }
                    if (found) {
                        var parent = el.closest('[role="article"]') || el.parentElement;
                        if (parent) {
                            parent.style.display = 'none';
                            parent.setAttribute('data-fbs10-sug', 'true');
                        }
                    }
                }
            })();""".trimIndent()
        webView.evaluateJavascript(js, null)
    }

    private fun injectTheme() {
        val css = when (currentTheme) {
            "dim" -> "body{background:#18191A!important;color:#E4E6EB!important;} div[role=article]{background:#242526!important;} input,textarea{background:#3a3b3c!important;color:#E4E6EB!important;} button{background:#31a24c!important;} a{color:#E7F3FF!important;}"
            "amoled" -> "body{background:#000000!important;color:#FFFFFF!important;} div[role=article]{background:#121212!important;} input,textarea{background:#1a1a1a!important;color:#FFFFFF!important;} button{background:#31a24c!important;} a{color:#E7F3FF!important;}"
            else -> ""
        }
        val js = if (css.isEmpty()) {
            "(function(){ var s=document.getElementById('fbs10-theme'); if(s) s.remove(); })();"
        } else {
            """(function() {
                    var s = document.getElementById('fbs10-theme');
                    if (!s) { s = document.createElement('style'); s.id = 'fbs10-theme'; document.head.appendChild(s); }
                    s.innerHTML = '$css';
                })();""".trimIndent()
        }
        webView.evaluateJavascript(js, null)
    }

    private fun expandComments() {
        val js = """(function() {
                var loaders = document.querySelectorAll('[role="button"]');
                for (var i = 0; i < loaders.length; i++) {
                    var el = loaders[i];
                    if (el.innerText && (el.innerText.includes('عرض المزيد') || el.innerText.includes('View more'))) {
                        el.click();
                    }
                }
            })();""".trimIndent()
        webView.evaluateJavascript(js, null)
    }

    private fun fetchBadges() {
        val jsNotif = """(function() {
                var count = 0;
                var badge = document.querySelector('[aria-label*="إشعارات"] [role="presentation"]') || document.querySelector('[data-visualcompletion="count"]');
                if (badge) {
                    var txt = badge.innerText || badge.textContent || '';
                    var num = parseInt(txt.replace(/\D/g,''));
                    if (!isNaN(num)) count = num;
                }
                return count;
            })();""".trimIndent()
        webView.evaluateJavascript(jsNotif) { value ->
            try {
                val num = value.replace("\"", "").toIntOrNull() ?: 0
                if (num > 0) {
                    badgeNotifications.text = num.toString()
                    badgeNotifications.visibility = View.VISIBLE
                } else {
                    badgeNotifications.visibility = View.GONE
                }
            } catch (e: Exception) {
                badgeNotifications.visibility = View.GONE
            }
        }

        val jsMsg = """(function() {
                var count = 0;
                var badge = document.querySelector('[aria-label*="Messenger"] [role="presentation"]') || document.querySelector('[aria-label*="الرسائل"] [role="presentation"]');
                if (badge) {
                    var txt = badge.innerText || badge.textContent || '';
                    var num = parseInt(txt.replace(/\D/g,''));
                    if (!isNaN(num)) count = num;
                }
                return count;
            })();""".trimIndent()
        webView.evaluateJavascript(jsMsg) { value ->
            try {
                val num = value.replace("\"", "").toIntOrNull() ?: 0
                if (num > 0) {
                    badgeMessages.text = num.toString()
                    badgeMessages.visibility = View.VISIBLE
                } else {
                    badgeMessages.visibility = View.GONE
                }
            } catch (e: Exception) {
                badgeMessages.visibility = View.GONE
            }
        }
    }

    private fun getBestFacebookUrl(): String {
        val cm = getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
        val caps = cm.getNetworkCapabilities(cm.activeNetwork)
        val slow = caps?.let { it.linkDownstreamBandwidthKbps < 600 } ?: true
        return if (slow) "https://mbasic.facebook.com/?paipv=0" else "https://m.facebook.com/?paipv=0"
    }

    override fun onBackPressed() {
        if (webView.canGoBack()) webView.goBack() else super.onBackPressed()
    }

    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)
        webView.saveState(outState)
    }

    override fun onRestoreInstanceState(savedInstanceState: Bundle) {
        super.onRestoreInstanceState(savedInstanceState)
        webView.restoreState(savedInstanceState)
    }
}
