# SQLi_payloads
This repo contains info about different types of SQL injections and their respective payloads with python scripts to automate some requests 
# Python Requests Setup

## 1. Installation

Install the `requests` library using pip:

```bash
python -m pip install requests
```

## 2. Import and Basic Usage

Import `requests` and send a GET request to your local test application.

```python
import requests

url = "http://127.0.0.1:5000/"

response = requests.get(url, timeout=10)

print("Status Code:", response.status_code)
print("Response Length:", len(response.text))
```

## 3. Sending a POST Request

Use POST requests when the application accepts form data through the request body.

```python
import requests

url = "http://127.0.0.1:5000/"

data = {
    "username": "test",
    "password": "test"
}

response = requests.post(
    url,
    data=data,
    timeout=10
)

print("Status Code:", response.status_code)
print("Response Length:", len(response.text))
```

## 4. Reusable HTTP Request Template

This template allows you to configure the URL, HTTP method, and input parameter for an authorized lab or local test application.

```python
import requests

# Target application
URL = "http://127.0.0.1:5000/"

# HTTP method: GET or POST
METHOD = "GET"

# Application input parameters
PARAMS = {
    "id": "1"
}


def send_request(test_input):
    params = PARAMS.copy()
    params["id"] = test_input

    try:
        if METHOD.upper() == "GET":
            response = requests.get(
                URL,
                params=params,
                timeout=10
            )

        elif METHOD.upper() == "POST":
            response = requests.post(
                URL,
                data=params,
                timeout=10
            )

        else:
            raise ValueError("Unsupported HTTP method")

        print("Status Code:", response.status_code)
        print("Response Length:", len(response.text))

        return response

    except requests.RequestException as error:
        print("Request failed:", error)
        return None


if __name__ == "__main__":
    test_input = input("Enter test input: ")
    send_request(test_input)
```

## 5. Important Notes

- Replace the example URL with your authorized lab or local application.
- Change `PARAMS` to match the application's actual input fields.
- Use `response.status_code` to inspect the HTTP status code.
- Use `response.text` to inspect the response body.
- Use `len(response.text)` to measure the response length.
- A status code of `200` does not automatically mean that a SQL condition is true.
- Handle timeouts and request exceptions when automating tests.

**Scope:** Use these scripts only against applications you own or have explicit permission to test.
