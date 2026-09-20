package com.lagradost.cloudstream3.providers

import com.lagradost.cloudstream3.*
import com.lagradost.cloudstream3.utils.*
import org.jsoup.nodes.Element

class HenaoJaraProvider : MainAPI() {
    override var mainUrl = "https://henaojara.com"
    override var name = "HenaoJara"
    override val hasMainPage = true
    override var lang = "es"
    override val hasDownloadSupport = false
    override val supportedTypes = setOf(TvType.Anime, TvType.ONA, TvType.AnimeMovie)

    override val mainPage = mainPageOf(
        "$mainUrl/" to "Episodios nuevos",
        "$mainUrl/genero/accion/" to "Accion",
        "$mainUrl/genero/comedia/" to "Comedia",
        "$mainUrl/genero/drama/" to "Drama",
        "$mainUrl/genero/ciencia-ficcion/" to "Ciencia Ficción",
        "$mainUrl/genero/aventura/" to "Aventura"
    )

    override suspend fun getMainPage(page: Int, request: MainPageRequest): HomePageResponse {
        val isHome = request.name == "Episodios nuevos"
        val url = if (isHome || page <= 1) request.data else "${request.data}${if (request.data.contains("?")) "&" else "?"}pag=$page"
        val doc = app.get(url).document
        val items = doc.select("ul li").mapNotNull { it.toEpisodePageResult() }
        return newHomePageResponse(request.name, items, !isHome && items.isNotEmpty())
    }

    override suspend fun search(query: String, page: Int): SearchResponseList {
        val url = "$mainUrl/?s=$query${if (page > 1) "&pag=$page" else ""}"
        val doc = app.get(url).document
        val results = doc.select("ul li").mapNotNull { it.toEpisodePageResult() }
        return newSearchResponseList(results, results.isNotEmpty())
    }

    private fun Element.toEpisodePageResult(): SearchResponse? {
        val titleEl = selectFirst("h3.h a") ?: selectFirst("h3 a") ?: return null
        val title = titleEl.text().trim()
        if (title.isBlank()) return null
        val ep = selectFirst("b.e")?.text()?.trim() ?: ""
        val hrefRaw = selectFirst(".mepli.ls a")?.attr("href")
            ?: selectFirst("li.ls a")?.attr("href")
            ?: selectFirst("a")?.attr("href")
            ?: return null
        val href = fixUrlNull(hrefRaw) ?: return null
        val poster = selectFirst("img")?.attr("src") ?: selectFirst("img")?.attr("data-src")
        return newAnimeSearchResponse(title, href, TvType.Anime) {
            this.posterUrl = fixUrlNull(poster)
            if (ep.isNotBlank()) addSub(ep)
        }
    }

    override suspend fun load(url: String): LoadResponse? {
        val doc = app.get(url).document
        val title = doc.selectFirst("h1")?.text()?.trim() ?: doc.selectFirst("title")?.text()?.trim() ?: "Anime"
        val poster = doc.selectFirst("div.poster img")?.attr("src")
            ?: doc.selectFirst("img")?.attr("src")
        val plot = doc.selectFirst("div.sinopsis")?.text()
            ?: doc.selectFirst("div.description")?.text()
            ?: doc.selectFirst("p")?.text()

        val episodes = doc.select("ul li").mapNotNull { el ->
            val epName = el.selectFirst("b.e")?.text()?.trim()
                ?: el.selectFirst("h3 a")?.text()?.trim()
                ?: return@mapNotNull null
            val epUrl = fixUrlNull(el.selectFirst("a")?.attr("href")) ?: return@mapNotNull null
            newEpisode(epUrl) {
                name = epName
            }
        }.reversed()

        return newAnimeLoadResponse(title, url, TvType.Anime) {
            this.posterUrl = fixUrlNull(poster)
            this.plot = plot
            if (episodes.isNotEmpty()) {
                addEpisodes(DubStatus.Subbed, episodes)
            }
        }
    }

    override suspend fun loadLinks(
        data: String,
        isCasting: Boolean,
        subtitleCallback: (SubtitleFile) -> Unit,
        callback: (ExtractorLink) -> Unit
    ): Boolean {
        val doc = app.get(data).document
        // Busca iframes / reproductores
        doc.select("iframe").forEach { iframe ->
            val src = iframe.attr("src")
            if (src.isNotBlank()) {
                loadExtractor(fixUrl(src), data, subtitleCallback, callback)
            }
        }
        doc.select("div.player a, div.server a").forEach { a ->
            val href = a.attr("href")
            if (href.isNotBlank()) {
                loadExtractor(fixUrl(href), data, subtitleCallback, callback)
            }
        }
        return true
    }
}
