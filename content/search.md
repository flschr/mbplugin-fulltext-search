---
title: "Suche"
url: "/search/"
menu:
  main:
    name: "Suche"
    weight: 100
---

<div class="search-page">
  <div class="search-intro">
  {{< search_intro >}}
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
		resultsCount.textContent = '';
	}

	function renderResults(matches, keywords, query) {
		resultsList.innerHTML = '';
		if (matches.length === 0) {
			return;
		}

		matches.forEach(function(item) {
			var li = document.createElement('li');
			li.className = 'results-item';

			var article = document.createElement('article');
			article.className = 'result-card';

			var dateSpan = document.createElement('span');
			dateSpan.className = 'result-date';
			dateSpan.textContent = formatDate(item.date_published);

			var titleLink = document.createElement('a');
			titleLink.className = 'result-title';
			titleLink.href = item.url;
			titleLink.innerHTML = highlight(item.displayTitle, keywords);

			var headerLine = document.createElement('div');
			headerLine.className = 'result-header';

			if (item.displayTitle) {
				headerLine.appendChild(titleLink);
				var separator = document.createTextNode(' · ');
				headerLine.appendChild(separator);
			}

			headerLine.appendChild(dateSpan);

			var metaLine = null;
			if (item.metaString) {
				metaLine = document.createElement('p');
				metaLine.className = 'result-meta';
				metaLine.innerHTML = 'Stichworte: ' + highlight(item.metaString, keywords);
			}

			var snippet = document.createElement('p');
			snippet.className = 'result-snippet';
			snippet.innerHTML = buildSnippet(item.content, keywords);

			article.appendChild(headerLine);
			if (metaLine) {
				article.appendChild(metaLine);
			}
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
		if (matches.length === 0) {
			resultsCount.textContent = 'Keine Treffer für „' + trimmed + '“';
		} else {
			resultsCount.textContent = matches.length + ' ' + (matches.length === 1 ? 'Treffer' : 'Treffer') + ' für „' + trimmed + '“';
		}
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
				var tagsRaw = Array.isArray(item.tags) ? item.tags : [];
				var categoriesRaw = Array.isArray(item.categories) ? item.categories : [];
				var metaSet = Object.create(null);
				var metaValues = [];

				function appendMeta(values) {
					values.forEach(function(value) {
						var clean = normalizeText(value);
						if (!clean || metaSet[clean]) {
							return;
						}
						metaSet[clean] = true;
						metaValues.push(clean);
					});
				}

				appendMeta(tagsRaw);
				appendMeta(categoriesRaw);

				var metaString = metaValues.join(', ');

				return {
					id: item.id || item.url,
					url: item.url,
					date_published: item.date_published,
					displayTitle: title || 'Beitrag ohne Titel',
					content: content,
					metaValues: metaValues,
					metaString: metaString,
					searchText: (title + ' ' + content + ' ' + metaString).toLowerCase()
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
	margin: 0;
	padding: 0;
	max-width: none;
}

.search-intro p {
	margin-top: 0;
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
	border: 1px solid var(--border, #d4d4d4);
	border-radius: 0.75rem;
	background: var(--surface, #ffffff);
	color: inherit;
	transition: border-color 0.2s ease, box-shadow 0.2s ease;
}

.search-input[disabled] {
	cursor: wait;
	opacity: 0.7;
}

.clear-search {
	position: absolute;
	right: 0.85rem;
	top: 50%;
	transform: translateY(-65%);
	display: flex;
	align-items: center;
	justify-content: center;
	width: 1.75rem;
	height: 1.75rem;
	background: transparent;
	border: none;
	border-radius: 50%;
	cursor: pointer;
	padding: 0;
	font-size: 1.1rem;
	line-height: 1;
}

.search-loading {
	display: flex;
	align-items: center;
	justify-content: center;
}

.search-error {
	margin-bottom: 1rem;
	padding: 0.75rem 1rem;
	border-radius: 0.5rem;
	background: rgba(220, 38, 38, 0.1);
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
	justify-content: flex-start;
	gap: 0.75rem;
}

.results-count {
	margin: 0;
}

.results-list {
	list-style: none;
	margin: 0;
	padding: 0;
	display: flex;
	flex-direction: column;
	gap: 1.25rem;
}

.results-item {
	margin: 0;
}

.result-card {
	padding: 0;
	background: transparent;
	border: none;
	box-shadow: none;
}

.result-header {
	display: flex;
	align-items: baseline;
	gap: 0.35rem;
	margin-bottom: 0.35rem;
}

.result-header .result-date {
	margin: 0;
	font-weight: inherit;
	color: inherit;
}

.result-meta {
	margin: 0 0 0.35rem;
	color: inherit;
	opacity: 0.8;
}

.result-snippet {
	margin: 0;
	line-height: 1.5;
	color: inherit;
}

.result-snippet mark,
.result-meta mark {
	background: #ffd955;
	color: inherit;
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
		padding: 0;
	}
}
</style>
