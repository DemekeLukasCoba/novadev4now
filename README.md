public AzureMapsSearchController(IHttpClientFactory httpClientFactory)
    {
        _httpClientFactory = httpClientFactory;
    }

    [HttpGet("searchPOI")]
    public async Task<IActionResult> SearchPOIAsync(string query, double lat, double lon)
    {
        var httpClient = _httpClientFactory.CreateClient();
        string url = $"https://atlas.microsoft.com/search/poi/json?subscription-key={azureMapsSubscriptionKey}&api-version=1.0&query={query}&lat={lat}&lon={lon}&limit=10";

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
        var httpClient = _httpClientFactory.CreateClient();
        string url = $"https://atlas.microsoft.com/search/address/json?subscription-key={azureMapsSubscriptionKey}&api-version=1.0&query={query}&lat={lat}&lon={lon}&limit=10";

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
}
