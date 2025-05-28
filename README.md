<!DOCTYPE html>
<html lang="en">

<head>
    <title>Interactive Search Quickstart - Azure Maps Web SDK Samples</title>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no" />
    <meta name="description" content="This tutorial shows how to create an interactive search experience." />
    <meta name="keywords" content="Microsoft maps, map, gis, API, SDK, services, module, tutorials, search, point of interest, POI" />
    <meta name="author" content="Microsoft Azure Maps" />

    <style>
        /* CSS styles as before */
    </style>

    <script>
        var map, datasource, popup, searchInput, resultsPanel, searchInputLength, centerMapOnResults;

        // The minimum number of characters needed in the search input before a search is performed.
        var minSearchInputLength = 3;
        var keyStrokeDelay = 150;

        function getMap() {
            // Initialize a map instance.
            map = new atlas.Map('myMap', {
                center: [-118.270293, 34.039737],
                zoom: 14,
                view: 'Auto'
            });

            // Store a reference to the Search Info Panel.
            resultsPanel = document.getElementById("results-panel");

            // Add key up event to the search box.
            searchInput = document.getElementById("search-input");
            searchInput.addEventListener("keyup", searchInputKeyup);

            // Create a popup which we can reuse for each result.
            popup = new atlas.Popup();

            // Wait until the map resources are ready.
            map.events.add('ready', function () {
                // Add the zoom control to the map.
                map.controls.add(new atlas.control.ZoomControl(), {
                    position: 'top-right'
                });

                // Create a data source and add it to the map.
                datasource = new atlas.source.DataSource();
                map.sources.add(datasource);

                // Add a layer for rendering the results.
                var searchLayer = new atlas.layer.SymbolLayer(datasource, null, {
                    iconOptions: {
                        image: 'pin-round-darkblue',
                        anchor: 'center',
                        allowOverlap: true
                    }
                });
                map.layers.add(searchLayer);

                // Add a click event to the search layer and show a popup when a result is clicked.
                map.events.add("click", searchLayer, function (e) {
                    // Make sure the event occurred on a shape feature.
                    if (e.shapes && e.shapes.length > 0) {
                        showPopup(e.shapes[0]);
                    }
                });
            });
        }

        function searchInputKeyup(e) {
            centerMapOnResults = false;
            if (searchInput.value.length >= minSearchInputLength) {
                if (e.keyCode === 13) {
                    centerMapOnResults = true;
                    search('searchPOI');
                }
                // Wait 100ms and see if the input length is unchanged before performing a search.
                setTimeout(function () {
                    if (searchInputLength == searchInput.value.length) {
                        search('searchPOI');
                    }
                }, keyStrokeDelay);
            } else {
                resultsPanel.innerHTML = '';
            }
            searchInputLength = searchInput.value.length;
        }

        function search(type) {
            // Remove any previous results from the map.
            datasource.clear();
            popup.close();
            resultsPanel.innerHTML = '';

            var query = document.getElementById("search-input").value;
            var lat = map.getCamera().center[1];
            var lon = map.getCamera().center[0];
            var url;

            if (type === 'searchPOI') {
                url = `/api/AzureMapsSearch/searchPOI?query=${query}&lat=${lat}&lon=${lon}`;
            } else if (type === 'searchAddress') {
                url = `/api/AzureMapsSearch/searchAddress?query=${query}&lat=${lat}&lon=${lon}`;
            }

            fetch(url).then(response => response.json()).then(data => {
                // Continue processing results similar to previous JavaScript example
                var html = [];
                for (var i = 0; i < data.length; i++) {
                    var r = data[i];
                    html.push('<li onclick="itemClicked(\'', i, '\')" onmouseover="itemHovered(\'', i, '\')">');
                    html.push('<div class="title">');
                    html.push(r.name || r.freeformAddress);
                    html.push('</div><div class="info">', r.address || r.freeformAddress, '</div>');
                    html.push('</li>');
                }

                resultsPanel.innerHTML = html.join('');
            });
        }

        function itemHovered(index) {
            // Show a popup when hovering an item in the result list.
            var shape = datasource.getShapes()[index];
            showPopup(shape);
        }

        function itemClicked(index) {
            // Center the map over the clicked item from the result list.
            var shape = datasource.getShapes()[index];
            map.setCamera({
                center: shape.getCoordinates(),
                zoom: 17
            });
        }

        function showPopup(shape) {
            var properties = shape.getProperties();
            // Create the HTML content of the POI to show in the popup.
            var html = ['<div class="poi-box">'];
            html.push('<div class="poi-title-box"><b>');
            html.push(properties.name || properties.address);
            html.push('</b></div><div class="poi-content-box"><div class="info location">', properties.address, '</div>');

            if (properties.phone) {
                html.push('<div class="info phone">', properties.phone, '</div>');
            }
            if (properties.url) {
                html.push('<div><a class="info website" href="http://', properties.url, '">http://', properties.url, '</a></div>');
            }

            html.push('</div></div>');
            popup.setOptions({
                position: shape.getCoordinates(),
                content: html.join('')
            });
            popup.open(map);
        }
    </script>
</head>

<body onload="getMap()">
    <div id="myMap"></div>
    <div id="search">
        <div class="search-input-box">
            <div class="search-input-group">
                <div class="search-icon" type="button"></div>
                <input id="search-input" type="text" placeholder="Search">
            </div>
        </div>
        <ul id="results-panel"></ul>
    </div>
</body>

</html>
