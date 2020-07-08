---
title: "Search"
url: "/search/"
menu: "main"
weight: 100
---

<script language="javascript">

var archive_results = {};

function hideHeader() {
	var tags = document.getElementsByTagName("header");
	if (tags.length > 0) {
		tags[0].parentNode.removeChild(tags[0]);
	}
}

function downloadArchive() {
	var xmlhttp = new XMLHttpRequest();
	xmlhttp.onreadystatechange = function() {
		if (this.readyState == 4 && this.status == 200) {
			archive_results = JSON.parse(this.responseText);
		}
	};
	xmlhttp.open("GET", "/archive/index.json", true);
	xmlhttp.send();
}

function runSearch(q) {
	var results_node = document.getElementById("list_results");
	results_node.innerHTML = "";
	if (q.length > 0) {
		for (var i = 0; i < archive_results.items.length; i++) {
			var item = archive_results.items[i];
			var title_lower = item.title.toLowerCase();
			var text_lower = item.content_text.toLowerCase();
			if (title_lower.includes(q) || text_lower.includes(q)) {
				var p_node = document.createElement("p");        
				var link_node = document.createElement("a");
				var d = Date.parse(item.date_published);
				var date_s = new Date(d).toISOString().substr(0, 10);
				var date_node = document.createTextNode(date_s); 
				link_node.appendChild(date_node);
				link_node.href = item.url;
				var s;
				if (item.title.length > 0) {
					s = item.title + ": " + item.content_text;
				}
				else {
					s = item.content_text;
				}
				if (s.length > 200) {
					s = s.substr(0, 200) + "...";
				}
				var text_node = document.createTextNode(": " + s); 
				p_node.appendChild(link_node);
				p_node.appendChild(text_node);
				results_node.appendChild(p_node);
			}
		}
	} 
}

hideHeader();
downloadArchive();

</script>

<style>

#search {
	display: none;
}

.field {
	width: 270px;
	height: 34px;
	font-size: 13px;
	font-weight: 400;
	padding-left: 12px;
	border: 2px solid #eee;
	margin-top: 20px;
	border-radius: 17px;
	-webkit-appearance: none;
}

</style>

<form onSubmit="return false;">
	<input class="field" type="text" name="q" id="input_search" placeholder="Search" onChange="runSearch(this.value.toLowerCase());" />
</form>

<div id="list_results">
</ul>
