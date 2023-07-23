# Azure OpenAI - Function Calling to retrieve Stock Price

As of 20th of July 2023, Azure OpenAI now supports the use of Function Calling with GPT-3.5 and GPT-4 models. This repo contains Jupyter notebook to test functional calling first-hand by retrieving stock price from [Alpha Vantage API](https://www.alphavantage.co/).

### 0. Pre-requisites:

1. Installa the latest version of openai Python package and **openai.api_version** variable to July 2023 version or above;
2. Retrieve Azure OpenAI GPT model deployment name, API endpoint and API key and assign them to "**OPENAI_API_DEPLOY**", "**OPENAI_API_BASE**" and "**OPENAI_API_KEY**" environment variables;
3. Retrieve your Alpha Vantage API key and assign it to "**ALPHAVANTAGE_API_KEY**" environment variable.

### 1. Function to get stock price from Alpha Vantage API:

We'll use **get_stock_price** custom function to interact with stock API and retrieve the lowest and highest prices on specific date for the stock symbol of interest. JSON structure returned by API endpoint is parsed to extract only specific time series values.
``` Python
def get_stock_price(symbol, date):
    print(f"Getting stock price for {symbol} on {date}")
    try:
        stock_url = f"https://www.alphavantage.co/query?function=TIME_SERIES_DAILY&symbol={symbol}&apikey={av_api_key}"
        stock_data = requests.get(stock_url)
        stock_low = stock_data.json()["Time Series (Daily)"][date]["3. low"]
        stock_high = stock_data.json()["Time Series (Daily)"][date]["2. high"]
        return stock_low, stock_high
    except:
        return "NA", "NA"
```

### 2. JSON structure for GPT function definition:

Azure OpenAI GPT models v0613 were trained to understand function structure that contain "**name**" and "**parameters**" fields. In my example, I indicate that the mddel can extract and match 2 mandatory properties: stock symbol for requested company and date in _YYYY-MM-DD_ format.

``` JSON
functions = [
    {
        "name": "get_stock_price",
        "description": "Retrieve lowest and highest stock price for a given stock symbol and date",
        "parameters": {
            "type": "object",
            "properties": {
                "symbol": {
                    "type": "string",
                    "description": "Stock symbol, for example MSFT for Microsoft"
                },
                "date": {
                    "type": "string",
                    "description": "Date in YYYY-MM-DD format"
                }
            },
            "required": ["symbol", "date"],
        }   
    }
]
```

### 3. Helper function to check provided arguments:

I copied function "**check_args**" from [Azure Samples code](https://github.com/Azure-Samples/openai/tree/main/Basic_Samples/Functions) "as is". It helps to verify whether GPT model provided all requirements arguments / parameters, and also if it tried to submit ones which our function doesn't expect.

### 4. Helper function to interact with Azure OpenAI GPT model:

This helper function follows 3-step logic:
1. First, it submits user's prompts, informs GPT model about available functions and sets calling mode to "**auto**", so that the model automatically decides whether it wants to call a function based on the prompt details and matching function's capabilities
    ``` Python
    response = openai.ChatCompletion.create(
        deployment_id=aoai_deployment,
        messages=messages,
        functions=functions,
        function_call="auto", 
    )
    ```
2. Next, it checks what functions the model wanted to call, if any.
   ``` Python
    response_message = response["choices"][0]["message"]

    # Step 2: Check if GPT model wanted to call a function
    if response_message.get("function_call"):
        print("Recommended function call:")
        print(response_message.get("function_call"))
        print()
   ```
3. Finally, if there is a matching function and it passes previous help function's test on validity of its arguments, I call the target function with parameter values extracted from the user's prompt.
   ``` Python
   function_response1, function_response2 = function_to_call(function_args["symbol"], function_args["date"])
   function_response = f"Lowest stock price: {function_response1}, Highest stock price: {function_response2}"
   ```
