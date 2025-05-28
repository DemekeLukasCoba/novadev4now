@page
@model Novadev.Pages.IndexModel
@{
    ViewData["Title"] = "Home page";
}

<div class="container">
    <div class="intro-section">
        <h1>Find nearby banks and ATMs</h1>
        <p class="lead mb-4">Quickly locate the nearest banks in your area</p>
        <div class="search-bar">
            <label for="searchInput">Search for a bank or ATM:</label><br>
            <input type="text" id="searchInput" name="searchInput" class="form-control mb-2" placeholder="e.g. Komerční banka, Česká spořitelna...">
        </div>
    </div>
    <div id="result"></div>
    <div id="map" style="height: 500px;"></div>
</div>

@section Scripts {
    <script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>
    <link rel="stylesheet" href="https://code.jquery.com/ui/1.13.2/themes/base/jquery-ui.css" />
    <script src="https://code.jquery.com/ui/1.13.2/jquery-ui.min.js"></script>

    <script src="https://atlas.microsoft.com/sdk/js/atlas.min.js?api-version=2"></script>
    <link rel="stylesheet" href="https://atlas.microsoft.com/sdk/css/atlas.min.css?api-version=2" />

    <script>
        let map;
        let marker = null;

        function initializeMap() {
            map = new atlas.Map('map', {
                center: [0, 0],
                zoom: 2,
                authOptions: {
                    authType: 'subscriptionKey',
                    subscriptionKey: '9L98ONwzVRUxJoCBRC0eS7LsA2RnB7ghI0qZfeIt5aVCRZzDykbFJQQJ99BEAC5RqLJZjxS7AAAgAZMP2QkN'
                }
            });

            map.events.add('ready', function () {
                setupAutocomplete();
            });
        }

        function setupAutocomplete() {
            $("#searchInput").autocomplete({
                source: function (request, response) {
                    $.ajax({
                        url: '@Url.Page("Index", "SearchPoiAutocomplete")',
                        data: { term: request.term },
                        success: function (data) {
                            response(data);
                        },
                        error: function () {
                            response([]);
                        }
                    });
                },
                select: function (event, ui) {
                    const position = [ui.item.lon, ui.item.lat];
                    map.setCamera({ center: position, zoom: 12 });

                    if (marker) {
                        map.markers.remove(marker);
                    }

                    marker = new atlas.HtmlMarker({
                        position: position,
                        text: '📍'
                    });
                    map.markers.add(marker);

                    searchNearbyPOIs(ui.item.lat, ui.item.lon);
                },
                minLength: 2
            });
        }

        function searchNearbyPOIs(lat, lon) {
            $.getJSON('@Url.Page("Index", "SearchPOI")', { lat: lat, lon: lon }, function (data) {
                if (!data || data.length === 0) {
                    $('#result').html('<p>No nearby banks/ATMs found.</p>');
                    return;
                }

                $('#result').html('<ul class="list-group mb-2">' + data.map(poi =>
                    `<li class="list-group-item"><strong>${poi.name}</strong><br><small>${poi.address}</small></li>`
                ).join('') + '</ul>');

                data.forEach(poi => {
                    const poiMarker = new atlas.HtmlMarker({
                        position: [poi.lon, poi.lat],
                        color: 'blue',
                        text: '🏦'
                    });
                    map.markers.add(poiMarker);
                });
            });
        }

        $(document).ready(function () {
            initializeMap();
        });
    </script>
}

using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using Microsoft.Extensions.Logging;
using System.Net.Http;
using System.Threading.Tasks;
using System.Text.Json;
using System.Collections.Generic;

namespace Novadev.Pages
{
    public class IndexModel : PageModel
    {
        private readonly ILogger<IndexModel> _logger;
        private readonly IHttpClientFactory _httpClientFactory;

        public IndexModel(ILogger<IndexModel> logger, IHttpClientFactory httpClientFactory)
        {
            _logger = logger;
            _httpClientFactory = httpClientFactory;
        }

        // 🔄 Autocomplete POI (banks/ATMs)
        public async Task<IActionResult> OnGetSearchPoiAutocompleteAsync(string term)
        {
            if (string.IsNullOrWhiteSpace(term))
                return BadRequest("Search term is required.");

            var httpClient = _httpClientFactory.CreateClient();
            string subscriptionKey = "9L98ONwzVRUxJoCBRC0eS7LsA2RnB7ghI0qZfeIt5aVCRZzDykbFJQQJ99BEAC5RqLJZjxS7AAAgAZMP2QkN";
            string url = $"https://atlas.microsoft.com/search/poi/json?api-version=1.0&subscription-key={subscriptionKey}&query={term}&limit=5";

            var response = await httpClient.GetAsync(url);
            if (!response.IsSuccessStatusCode)
                return StatusCode((int)response.StatusCode, "Error fetching POI suggestions.");

            var json = await response.Content.ReadAsStringAsync();
            var data = JsonDocument.Parse(json);
            var results = data.RootElement.GetProperty("results");

            var suggestions = new List<object>();
            foreach (var result in results.EnumerateArray())
            {
                var poi = result.GetProperty("poi");
                var position = result.GetProperty("position");

                suggestions.Add(new
                {
                    label = poi.GetProperty("name").GetString(),
                    value = poi.GetProperty("name").GetString(),
                    lat = position.GetProperty("lat").GetDouble(),
                    lon = position.GetProperty("lon").GetDouble()
                });
            }

            return new JsonResult(suggestions);
        }

        // 🔎 Get nearby POIs (banks/ATMs) around selected location
        public async Task<IActionResult> OnGetSearchPOIAsync(double lat, double lon)
        {
            var httpClient = _httpClientFactory.CreateClient();
            string subscriptionKey = "9L98ONwzVRUxJoCBRC0eS7LsA2RnB7ghI0qZfeIt5aVCRZzDykbFJQQJ99BEAC5RqLJZjxS7AAAgAZMP2QkN";
            int radius = 5000;

            string url = $"https://atlas.microsoft.com/search/poi/category/json?api-version=1.0&subscription-key={subscriptionKey}&lat={lat}&lon={lon}&radius={radius}&categorySet=7323,7376";

            var response = await httpClient.GetAsync(url);
            if (!response.IsSuccessStatusCode)
                return StatusCode((int)response.StatusCode, "Error fetching POIs.");

            var json = await response.Content.ReadAsStringAsync();
            var data = JsonDocument.Parse(json);
            var results = data.RootElement.GetProperty("results");

            var pois = new List<object>();
            foreach (var result in results.EnumerateArray())
            {
                var position = result.GetProperty("position");
                var poi = result.GetProperty("poi");
                var address = result.GetProperty("address");

                pois.Add(new
                {
                    name = poi.GetProperty("name").GetString(),
                    lat = position.GetProperty("lat").GetDouble(),
                    lon = position.GetProperty("lon").GetDouble(),
                    address = address.GetProperty("freeformAddress").GetString()
                });
            }

            return new JsonResult(pois);
        }

        public void OnGet()
        {
        }
    }
}
