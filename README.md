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

        public async Task<IActionResult> OnGetSearchCityAsync(string term)
        {
            if (string.IsNullOrWhiteSpace(term))
                return BadRequest("Search term is required.");

            var client = _httpClientFactory.CreateClient();
            string key = "9L98ONwzVRUxJoCBRC0eS7LsA2RnB7ghI0qZfeIt5aVCRZzDykbFJQQJ99BEAC5RqLJZjxS7AAAgAZMP2QkN";
            string url = $"https://atlas.microsoft.com/search/address/json?api-version=1.0&subscription-key={key}&typeahead=true&query={term}&limit=5";

            var res = await client.GetAsync(url);
            if (!res.IsSuccessStatusCode)
                return StatusCode((int)res.StatusCode);

            var json = await res.Content.ReadAsStringAsync();
            var doc = JsonDocument.Parse(json);
            var suggestions = new List<object>();

            foreach (var result in doc.RootElement.GetProperty("results").EnumerateArray())
            {
                var address = result.GetProperty("address").GetProperty("freeformAddress").GetString();
                var lat = result.GetProperty("position").GetProperty("lat").GetDouble();
                var lon = result.GetProperty("position").GetProperty("lon").GetDouble();

                suggestions.Add(new { label = address, value = address, lat, lon });
            }

            return new JsonResult(suggestions);
        }

        public async Task<IActionResult> OnGetSearchBanksAsync(double lat, double lon)
        {
            var client = _httpClientFactory.CreateClient();
            var radius = 30000; // 30 km
            var queries = new[] { "bank", "atm" };
            var allItems = new List<object>();

            foreach (var query in queries)
            {
                string key = "9L98ONwzVRUxJoCBRC0eS7LsA2RnB7ghI0qZfeIt5aVCRZzDykbFJQQJ99BEAC5RqLJZjxS7AAAgAZMP2QkN";
                var url = $"https://atlas.microsoft.com/search/poi/json?api-version=1.0&subscription-key={key}&query={query}&lat={lat}&lon={lon}&radius={radius}"; 
            } }
    
        public AzureMapsSearchController(IHttpClientFactory httpClientFactory)
        {
            _httpClientFactory = httpClientFactory;
        }

        [HttpGet("searchPOI")]
        public async Task<IActionResult> SearchPOIAsync(string query, double lat, double lon)
        {
            string key = "9L98ONwzVRUxJoCBRC0eS7LsA2RnB7ghI0qZfeIt5aVCRZzDykbFJQQJ99BEAC5RqLJZjxS7AAAgAZMP2QkN";
            var httpClient = _httpClientFactory.CreateClient();
            string url = $"https://atlas.microsoft.com/search/poi/json?subscription-key={key}&api-version=1.0&query={query}&lat={lat}&lon={lon}&limit=10";

            var response = await httpClient.GetAsync(url);
            if (!response.IsSuccessStatusCode)
            {
                return StatusCode((int)response.StatusCode, "Error fetching POIs.");
            }

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

        [HttpGet("searchAddress")]
        public async Task<IActionResult> SearchAddressAsync(string query, double lat, double lon)
        {
            string key = "9L98ONwzVRUxJoCBRC0eS7LsA2RnB7ghI0qZfeIt5aVCRZzDykbFJQQJ99BEAC5RqLJZjxS7AAAgAZMP2QkN";
            var httpClient = _httpClientFactory.CreateClient();
            string url = $"https://atlas.microsoft.com/search/address/json?subscription-key={key}&api-version=1.0&query={query}&lat={lat}&lon={lon}&limit=10";

            var response = await httpClient.GetAsync(url);
            if (!response.IsSuccessStatusCode)
            {
                return StatusCode((int)response.StatusCode, "Error fetching addresses.");
            }

            var json = await response.Content.ReadAsStringAsync();
            var data = JsonDocument.Parse(json);
            var results = data.RootElement.GetProperty("results");

            var addresses = new List<object>();
            foreach (var result in results.EnumerateArray())
            {
                var position = result.GetProperty("position");
                var address = result.GetProperty("address");

                addresses.Add(new
                {
                    freeformAddress = address.GetProperty("freeformAddress").GetString(),
                    lat = position.GetProperty("lat").GetDouble(),
                    lon = position.GetProperty("lon").GetDouble()
                });
            }

            return new JsonResult(addresses);
        }




        public void OnGet() { }
    }
}
