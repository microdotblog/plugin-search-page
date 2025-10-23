---
title: "Search"
url: "/search/"
menu: "main"
weight: 100
---

<div class="search-page">
	<div class="search-intro">
		<p>Finde Beiträge im Blog. Ergebnisse erscheinen bereits beim Tippen.</p>
	</div>

	<form id="searchForm" class="search-form" role="search">
		<div class="search-box">
			<input
				type="search"
				name="q"
				id="searchInput"
				class="search-input"
				placeholder="Suchbegriff eingeben…"
				autocomplete="off"
				disabled
				aria-label="Blog durchsuchen">
			<button type="button" id="clearSearch" class="clear-search is-hidden" aria-label="Eingabe löschen">×</button>
		</div>
	</form>

	<div id="searchLoading" class="search-loading is-hidden">
		<span>Lade Archiv…</span>
	</div>

	<div id="searchError" class="search-error is-hidden" role="alert"></div>

	<div id="resultsContainer" class="results-container is-hidden">
		<div class="results-meta">
			<p id="resultsCount" class="results-count"></p>
		</div>
		<p id="resultsEmpty" class="results-empty is-hidden"></p>
		<ul id="resultsList" class="results-list"></ul>
	</div>
</div>

<script>
(function() {
	'use strict';

	var searchInput = document.getElementById('searchInput');
	var clearButton = document.getElementById('clearSearch');
	var searchForm = document.getElementById('searchForm');
	var resultsContainer = document.getElementById('resultsContainer');
	var resultsCount = document.getElementById('resultsCount');
	var resultsList = document.getElementById('resultsList');
	var resultsEmpty = document.getElementById('resultsEmpty');
	var searchLoading = document.getElementById('searchLoading');
	var searchError = document.getElementById('searchError');

	var archiveItems = [];
	var loadingTimer = null;

	function normalizeText(value) {
		return (value || '').replace(/\s+/g, ' ').trim();
	}

	function escapeRegExp(value) {
		return value.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
	}

	function highlight(text, keywords) {
		var highlighted = text;
		keywords.forEach(function(keyword) {
			if (!keyword) {
				return;
			}
			var pattern = new RegExp('(' + escapeRegExp(keyword) + ')', 'ig');
			highlighted = highlighted.replace(pattern, '<mark>$1</mark>');
		});
		return highlighted;
	}

	function buildSnippet(text, keywords) {
		if (!text) {
			return '';
		}
		var lowerText = text.toLowerCase();
		var snippetLength = 220;
		var firstIndex = -1;
		for (var i = 0; i < keywords.length; i += 1) {
			var keyword = keywords[i];
			if (!keyword) {
				continue;
			}
			var idx = lowerText.indexOf(keyword.toLowerCase());
			if (idx !== -1) {
				firstIndex = idx;
				break;
			}
		}
		if (firstIndex === -1) {
			firstIndex = 0;
		}
		var start = Math.max(0, firstIndex - 40);
		var end = Math.min(text.length, start + snippetLength);
		var excerpt = text.slice(start, end);
		if (start > 0) {
			excerpt = '…' + excerpt;
		}
		if (end < text.length) {
			excerpt += '…';
		}
		return highlight(excerpt, keywords);
	}

	function formatDate(value) {
		if (!value) {
			return '';
		}
		var date = new Date(value);
		if (isNaN(date.getTime())) {
			return '';
		}
		try {
			return date.toLocaleDateString('de-DE', {
				year: 'numeric',
				month: '2-digit',
				day: '2-digit'
			});
		} catch (err) {
			var year = date.getFullYear();
			var month = String(date.getMonth() + 1).padStart(2, '0');
			var day = String(date.getDate()).padStart(2, '0');
			return day + '.' + month + '.' + year;
		}
	}

	function toggleClearButton(value) {
		if (value && value.trim().length > 0) {
			clearButton.classList.remove('is-hidden');
		} else {
			clearButton.classList.add('is-hidden');
		}
	}

	function hideResults() {
		resultsContainer.classList.add('is-hidden');
		resultsList.innerHTML = '';
		resultsEmpty.classList.add('is-hidden');
		resultsCount.textContent = '';
	}

	function renderResults(matches, keywords, query) {
		resultsList.innerHTML = '';
		if (matches.length === 0) {
			resultsEmpty.textContent = 'Kein Ergebnis für „' + query + '“.';
			resultsEmpty.classList.remove('is-hidden');
			return;
		}
		resultsEmpty.classList.add('is-hidden');

		matches.forEach(function(item) {
			var li = document.createElement('li');
			li.className = 'results-item';

			var article = document.createElement('article');
			article.className = 'result-card';

			var meta = document.createElement('div');
			meta.className = 'result-meta';

			var dateSpan = document.createElement('span');
			dateSpan.textContent = formatDate(item.date_published);
			meta.appendChild(dateSpan);

			var tagSource = item.tags.length > 0 ? item.tags : item.categories;
			if (tagSource && tagSource.length > 0) {
				var tagList = document.createElement('div');
				tagList.className = 'result-tags';
				tagSource.forEach(function(rawTag) {
					var cleanTag = normalizeText(rawTag);
					if (!cleanTag) {
						return;
					}
					var tagButton = document.createElement('button');
					tagButton.type = 'button';
					tagButton.className = 'result-tag';
					tagButton.textContent = cleanTag;
					tagButton.addEventListener('click', function() {
						searchInput.value = cleanTag;
						submitSearch(cleanTag);
						searchInput.focus();
					});
					tagList.appendChild(tagButton);
				});
				if (tagList.childNodes.length > 0) {
					var separator = document.createTextNode(' · ');
					meta.appendChild(separator);
					meta.appendChild(tagList);
				}
			}

			var titleLink = document.createElement('a');
			titleLink.className = 'result-title';
			titleLink.href = item.url;
			titleLink.textContent = item.displayTitle;

			var snippet = document.createElement('p');
			snippet.className = 'result-snippet';
			snippet.innerHTML = buildSnippet(item.content, keywords);

			article.appendChild(meta);
			article.appendChild(titleLink);
			article.appendChild(snippet);
			li.appendChild(article);
			resultsList.appendChild(li);
		});
	}

	function runSearch(query) {
		var currentValue = query || '';
		var trimmed = currentValue.trim();

		toggleClearButton(currentValue);

		if (trimmed.length === 0) {
			hideResults();
			return;
		}

		var keywords = trimmed.toLowerCase().split(/\s+/).filter(Boolean);
		if (keywords.length === 0) {
			hideResults();
			return;
		}

		var matches = archiveItems.filter(function(item) {
			return keywords.every(function(keyword) {
				return item.searchText.indexOf(keyword) !== -1;
			});
		});

		resultsContainer.classList.remove('is-hidden');
		resultsCount.textContent = matches.length + ' ' + (matches.length === 1 ? 'Treffer' : 'Treffer') + ' für „' + trimmed + '“';
		renderResults(matches, keywords, trimmed);
	}

	function submitSearch(query) {
		runSearch(query);
		var url = new URL(window.location.href);
		var trimmed = (query || '').trim();
		if (trimmed.length > 0) {
			url.searchParams.set('q', trimmed);
		} else {
			url.searchParams.delete('q');
		}
		history.replaceState({}, '', url);
	}

	function restoreInitialSearch() {
		var params = new URLSearchParams(window.location.search);
		var q = params.get('q');
		if (q && q.trim().length > 0) {
			searchInput.value = q;
			submitSearch(q);
		} else {
			toggleClearButton('');
		}
	}

	function finishLoading() {
		if (loadingTimer) {
			window.clearTimeout(loadingTimer);
			loadingTimer = null;
		}
		searchLoading.classList.add('is-hidden');
		searchInput.disabled = false;
		searchInput.focus();
	}

	searchForm.addEventListener('submit', function(event) {
		event.preventDefault();
		submitSearch(searchInput.value);
	});

	searchInput.addEventListener('input', function(event) {
		submitSearch(event.target.value);
	});

	searchInput.addEventListener('keydown', function(event) {
		if (event.key === 'Escape') {
			searchInput.value = '';
			submitSearch('');
		}
	});

	clearButton.addEventListener('click', function() {
		searchInput.value = '';
		submitSearch('');
		searchInput.focus();
	});

	loadingTimer = window.setTimeout(function() {
		searchLoading.classList.remove('is-hidden');
	}, 1500);

	fetch('/archive/index.json')
		.then(function(response) {
			if (!response.ok) {
				throw new Error('Archiv konnte nicht geladen werden (Status ' + response.status + ').');
			}
			return response.json();
		})
		.then(function(data) {
			var items = Array.isArray(data.items) ? data.items : [];
			archiveItems = items.map(function(item) {
				var title = normalizeText(item.title || '');
				var content = normalizeText(item.content_text || '');
				var tags = Array.isArray(item.tags) ? item.tags : [];
				var categories = Array.isArray(item.categories) ? item.categories : [];
				return {
					id: item.id || item.url,
					url: item.url,
					date_published: item.date_published,
					displayTitle: title || 'Beitrag ohne Titel',
					content: content,
					tags: tags,
					categories: categories,
					searchText: (title + ' ' + content).toLowerCase()
				};
			});
			finishLoading();
			restoreInitialSearch();
		})
		.catch(function(error) {
			finishLoading();
			searchError.textContent = 'Fehler beim Laden des Archivs: ' + error.message;
			searchError.classList.remove('is-hidden');
			searchInput.disabled = true;
			clearButton.disabled = true;
		});
})();
</script>

<style>
.search-page {
	max-width: 760px;
	margin: 0 auto;
	padding: 1rem;
}

.search-intro p {
	margin-top: 0;
	color: var(--accent2, #555);
}

.search-form {
	margin-bottom: 1.5rem;
}

.search-box {
	position: relative;
	display: flex;
	align-items: center;
}

.search-input {
	width: 100%;
	padding: 0.85rem 1.1rem;
	font-size: 1rem;
	border: 1px solid var(--border, #d4d4d4);
	border-radius: 0.75rem;
	background: var(--surface, #ffffff);
	color: inherit;
	transition: border-color 0.2s ease, box-shadow 0.2s ease;
}

.search-input:focus {
	outline: none;
	border-color: var(--link, #3b82f6);
	box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.25);
}

.search-input[disabled] {
	cursor: wait;
	opacity: 0.7;
}

.clear-search {
	position: absolute;
	right: 0.85rem;
	background: transparent;
	border: none;
	color: var(--accent2, #666);
	font-size: 1.5rem;
	line-height: 1;
	cursor: pointer;
	padding: 0;
}

.clear-search:hover,
.clear-search:focus {
	color: var(--link, #3b82f6);
}

.search-loading {
	display: flex;
	align-items: center;
	justify-content: center;
	font-size: 0.95rem;
	color: var(--accent2, #555);
}

.search-error {
	margin-bottom: 1rem;
	padding: 0.75rem 1rem;
	border-radius: 0.5rem;
	background: rgba(220, 38, 38, 0.1);
	color: #991b1b;
}

.results-container {
	display: flex;
	flex-direction: column;
	gap: 1rem;
}

.results-meta {
	display: flex;
	flex-wrap: wrap;
	align-items: center;
	justify-content: space-between;
	gap: 0.75rem;
}

.results-count {
	margin: 0;
	font-weight: 600;
	color: var(--accent2, #333);
}

.results-empty {
	margin: 0;
	font-style: italic;
	color: var(--accent2, #666);
}

.results-list {
	list-style: none;
	margin: 0;
	padding: 0;
	display: flex;
	flex-direction: column;
	gap: 1rem;
}

.results-item {
	margin: 0;
}

.result-card {
	padding: 1.1rem 1.2rem;
	background: var(--surface-elevated, rgba(0, 0, 0, 0.02));
	border-radius: 0.75rem;
	border: 1px solid rgba(0, 0, 0, 0.05);
	transition: border-color 0.2s ease, transform 0.2s ease, box-shadow 0.2s ease;
}

.result-card:hover {
	border-color: var(--link, #3b82f6);
	transform: translateY(-2px);
	box-shadow: 0 8px 20px rgba(0, 0, 0, 0.08);
}

.result-meta {
	font-size: 0.85rem;
	color: var(--accent2, #666);
	display: flex;
	flex-wrap: wrap;
	gap: 0.5rem;
	align-items: center;
	margin-bottom: 0.4rem;
}

.result-tags {
	display: flex;
	flex-wrap: wrap;
	gap: 0.35rem;
}

.result-tag {
	border: 1px solid var(--link, #3b82f6);
	background: transparent;
	color: var(--link, #3b82f6);
	border-radius: 999px;
	padding: 0.15rem 0.6rem;
	font-size: 0.75rem;
	cursor: pointer;
	transition: all 0.2s ease;
}

.result-tag:hover,
.result-tag:focus {
	background: var(--link, #3b82f6);
	color: var(--button-text, #fff);
}

.result-title {
	display: inline-block;
	font-size: 1.1rem;
	font-weight: 600;
	color: var(--link, #1f2937);
	text-decoration: none;
	margin-bottom: 0.45rem;
}

.result-title:hover,
.result-title:focus {
	text-decoration: underline;
}

.result-snippet {
	margin: 0;
	font-size: 0.95rem;
	line-height: 1.5;
	color: inherit;
}

.result-snippet mark {
	background: rgba(59, 130, 246, 0.2);
	padding: 0 0.15rem;
	border-radius: 0.2rem;
}

.is-hidden {
	display: none !important;
}

@media (max-width: 600px) {
	.search-page {
		padding: 0;
	}

	.result-card {
		padding: 1rem;
	}
}
</style>
