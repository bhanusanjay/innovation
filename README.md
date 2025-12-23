Yes, there are elegant ways to centralize this logging in both Spring Boot and Python without modifying your business logic extensively.

## Spring Boot Approach

**Use a `ClientHttpRequestInterceptor`** to intercept all outgoing HTTP requests matching your LLM API patterns:

```java
@Component
public class LLMloggingInterceptor implements ClientHttpRequestInterceptor {
    
    private static final Logger log = LoggerFactory.getLogger(LLMLoggingInterceptor.class);
    
    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body, 
                                        ClientHttpRequestExecution execution) throws IOException {
        
        URI uri = request.getURI();
        
        // Check if this is an LLM API call (adjust pattern as needed)
        if (uri.toString().contains("/chat/completions") || 
            uri.toString().contains("/completions")) {
            
            // Parse request body to log tokens sent
            ObjectMapper mapper = new ObjectMapper();
            JsonNode requestJson = mapper.readTree(body);
            
            long startTime = System.currentTimeMillis();
            ClientHttpResponse response = execution.execute(request, body);
            long duration = System.currentTimeMillis() - startTime;
            
            // Read and log response (need to wrap to allow re-reading)
            String responseBody = new String(response.getBody().readAllBytes(), StandardCharsets.UTF_8);
            JsonNode responseJson = mapper.readTree(responseBody);
            
            // Extract token usage
            JsonNode usage = responseJson.get("usage");
            if (usage != null) {
                log.info("LLM API Call - URL: {}, Model: {}, Prompt Tokens: {}, Completion Tokens: {}, Total Tokens: {}, Duration: {}ms",
                    uri,
                    requestJson.get("model").asText(),
                    usage.get("prompt_tokens").asInt(),
                    usage.get("completion_tokens").asInt(),
                    usage.get("total_tokens").asInt(),
                    duration
                );
            }
            
            // Return wrapped response that can be read again
            return new BufferingClientHttpResponseWrapper(response, responseBody);
        }
        
        return execution.execute(request, body);
    }
}
```

**Configure your RestTemplate/WebClient:**

```java
@Configuration
public class RestTemplateConfig {
    
    @Autowired
    private LLMLoggingInterceptor llmLoggingInterceptor;
    
    @Bean
    public RestTemplate restTemplate() {
        RestTemplate restTemplate = new RestTemplate();
        restTemplate.getInterceptors().add(llmLoggingInterceptor);
        return restTemplate;
    }
}
```

For **WebClient** (reactive), use an `ExchangeFilterFunction`:

```java
@Bean
public WebClient webClient() {
    return WebClient.builder()
        .filter(llmLoggingFilter())
        .build();
}

private ExchangeFilterFunction llmLoggingFilter() {
    return ExchangeFilterFunction.ofRequestProcessor(clientRequest -> {
        // Similar logging logic
        return Mono.just(clientRequest);
    });
}
```

## Python Approach

**Use an HTTP client wrapper or middleware** depending on your library:

### For `requests` library:

```python
import requests
import json
import logging
from requests.adapters import HTTPAdapter
from requests.models import Response

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class LLMLoggingAdapter(HTTPAdapter):
    def send(self, request, **kwargs):
        response = super().send(request, **kwargs)
        
        # Check if this is an LLM API call
        if '/chat/completions' in request.url or '/completions' in request.url:
            try:
                request_body = json.loads(request.body)
                response_body = json.loads(response.text)
                
                usage = response_body.get('usage', {})
                logger.info(
                    f"LLM API Call - URL: {request.url}, "
                    f"Model: {request_body.get('model')}, "
                    f"Prompt Tokens: {usage.get('prompt_tokens')}, "
                    f"Completion Tokens: {usage.get('completion_tokens')}, "
                    f"Total Tokens: {usage.get('total_tokens')}"
                )
            except Exception as e:
                logger.error(f"Error logging LLM metrics: {e}")
        
        return response

# Configure session with the adapter
session = requests.Session()
session.mount('https://', LLMLoggingAdapter())
session.mount('http://', LLMLoggingAdapter())

# Use this session throughout your app
response = session.post(url, json=data)
```

### For `httpx` library (modern alternative):

```python
import httpx
import json
import logging

logger = logging.getLogger(__name__)

class LLMLoggingTransport(httpx.HTTPTransport):
    def handle_request(self, request):
        response = super().handle_request(request)
        
        if b'/chat/completions' in request.url.raw or b'/completions' in request.url.raw:
            try:
                request_body = json.loads(request.content)
                response_body = json.loads(response.content)
                
                usage = response_body.get('usage', {})
                logger.info(
                    f"LLM API Call - URL: {request.url}, "
                    f"Model: {request_body.get('model')}, "
                    f"Tokens: {usage.get('total_tokens')}"
                )
            except Exception as e:
                logger.error(f"Error logging LLM metrics: {e}")
        
        return response

# Use custom transport
client = httpx.Client(transport=LLMLoggingTransport())
```

## Additional Recommendations

1. **Store metrics in a database or metrics system** (Prometheus, CloudWatch, etc.) instead of just logging
2. **Use AOP (Aspect-Oriented Programming)** in Spring Boot to intercept methods making LLM calls if you have a service layer pattern
3. **Consider using OpenTelemetry** for distributed tracing across both Java and Python services
4. **Add request/response caching** to reduce redundant API calls and costs

These approaches let you transparently log all LLM usage without touching your business logic, and you can easily extend them to track costs, errors, or other metrics.
