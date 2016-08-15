---
layout: post
title: Logging http request and response with Spring 4
---

Often we are faced with capturing http requests and responses for logging or other purposes. 
The main issue with reading request is that as soon as the input stream is consumed its gone whoof... and cannot be read again.
So the input stream has to be cached. Instead of writing your own classes for caching (which can be found at several places on web), Spring provides a couple of useful classes i.e. ContentCachingRequestWrapper and ContentCachingResponseWrapper.
These classes can be utilized very effectively, for example, in filters for logging purposes.

Define a filter in web.xml:

<filter>
    <filter-name>loggingFilter</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
</filter>
<filter-mapping>
    <filter-name>loggingFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>

Since the filter is declared as DelegatingFilterProxy, it can be declared as a bean using @Component or @Bean annotations.
In the loggingFilter's doFilter method, wrap the request and response with spring provided classes before passing it to the filter chain:

HttpServletRequest requestToCache = new ContentCachingRequestWrapper(request);
HttpServletResponse responseToCache = new ContentCachingResponseWrapper(response);
chain.doFilter(requestToCache, responseToCache);
String requestData = getRequestData(requestToCache);
String responseData = getResponseData(responseToCache);

The input stream will be cached in the wrapped request as soon as the input stream is consumed after chain.doFilter(). Then it can be accessed as below:

public static String getRequestData(final HttpServletRequest request) throws UnsupportedEncodingException {
    String payload = null;
    ContentCachingRequestWrapper wrapper = WebUtils.getNativeRequest(request, ContentCachingRequestWrapper.class);
    if (wrapper != null) {
        byte[] buf = wrapper.getContentAsByteArray();
        if (buf.length > 0) {
            payload = new String(buf, 0, buf.length, wrapper.getCharacterEncoding());
        }
    }
    return payload;
}

However, things are a bit different for response. Since the response was also wrapped before passing it to the filter chain, it will also be cached to the output stream as soon as it is written on its way back. But since the output stream will also be consumed so you have to copy the response back to the output stream using wrapper.copyBodyToResponse(). See below:

public static String getResponseData(final HttpServletResponse response) throws IOException {
    String payload = null;
    ContentCachingResponseWrapper wrapper =
        WebUtils.getNativeResponse(response, ContentCachingResponseWrapper.class);
    if (wrapper != null) {
        byte[] buf = wrapper.getContentAsByteArray();
        if (buf.length > 0) {
            payload = new String(buf, 0, buf.length, wrapper.getCharacterEncoding());
            wrapper.copyBodyToResponse();
        }
    }
    return payload;
}
