---
title: "Setting Up CloudFront Functions for Hugo Page Routing"
date: "2024-12-23"
description: "Learn how to configure CloudFront Functions to properly handle Hugo's page routing by redirecting /page/ to page/index.html"
categories: ["AWS", "Hugo", "CloudFront"]
tags: ["serverless", "static-site", "routing"]
---

# Setting Up CloudFront Functions for Hugo Page Routing

When hosting a Hugo site on AWS CloudFront, URLs ending with a trailing slash (like `/about/`) don't automatically resolve to their corresponding `index.html` files. While Lambda@Edge can solve this, CloudFront Functions provide a more cost-effective and faster solution. This article will guide you through setting up a CloudFront Function to handle this routing properly.

## Why CloudFront Functions?

CloudFront Functions are a perfect fit for this URL rewriting task because:

- **Cost**: Up to 1/6th the cost of Lambda@Edge
- **Performance**: Sub-millisecond execution times
- **Simplicity**: Easier to deploy and manage
- **Scale**: Automatic scaling at CloudFront's edge locations

## Creating the CloudFront Function

First, let's create the function that will handle the URL rewriting:

1. Go to the CloudFront console
2. Navigate to "Functions" in the left sidebar
3. Click "Create function"
4. Name your function (e.g., `hugo-page-routing`)
5. Use the following code:

```javascript
function handler(event) {
    var request = event.request;
    var uri = request.uri;
    
    // If URI ends with a trailing slash, append 'index.html'
    if (uri.endsWith('/')) {
        request.uri = uri + 'index.html';
    }
    // If URI doesn't have an extension, append '/index.html'
    else if (!uri.includes('.')) {
        request.uri = uri + '/index.html';
    }
    
    return request;
}
```

### Testing the Function

CloudFront provides a built-in testing environment:

1. Click the "Test" tab
2. Create test events for different scenarios:
   ```json
   {
     "request": {
       "uri": "/about/",
       "method": "GET"
     }
   }
   ```
3. Test cases should include:
   - `/about/` → `/about/index.html`
   - `/about` → `/about/index.html`
   - `/styles.css` → `/styles.css` (unchanged)
   - `/posts/my-post/` → `/posts/my-post/index.html`

## Associating with Your Distribution

After testing, associate the function with your CloudFront distribution:

1. Go to your CloudFront distribution
2. Select the "Behaviors" tab
3. Edit the default behavior
4. Under "Function associations":
   - Event Type: Viewer Request
   - Function: Select your function
5. Save changes

The deployment is immediate - no waiting for propagation like with Lambda@Edge!

## Function Limitations

Be aware of CloudFront Functions limitations:

- Maximum execution time: 1ms
- No external network calls
- No file system access
- Limited JavaScript runtime features
- Maximum function size: 10KB

For this URL rewriting task, these limitations don't affect us.

## Monitoring and Logs

Monitor your function's performance:

1. CloudFront console provides:
   - Invocation metrics
   - Error rates
   - Execution time statistics
2. CloudWatch metrics show:
   - Function invocations
   - Error count
   - Latency percentiles

## Common Issues and Solutions

### Cache Behavior Settings

Ensure your CloudFront cache behavior is configured correctly:

- Forward query strings if needed by Hugo
- Set appropriate TTLs
- Configure correct origin response timeout

### 403/404 Errors

If you encounter these errors:

1. Check S3 bucket permissions
2. Verify CloudFront origin path
3. Test function behavior with sample requests
4. Validate URI rewriting logic

### Function Execution Errors

If the function fails:

1. Check syntax errors
2. Verify URI manipulation logic
3. Test with various input patterns
4. Review CloudFront metrics

## Best Practices

1. **Testing**:
   - Test all URL patterns
   - Verify function behavior with query strings
   - Check handling of special characters

2. **Maintenance**:
   - Keep function code simple
   - Document URL transformation rules
   - Monitor performance metrics

3. **Security**:
   - Don't include sensitive information
   - Validate input URIs
   - Consider adding security headers

4. **Development**:
   - Use version control for function code
   - Maintain test cases
   - Document deployment process

## Cost Considerations

CloudFront Functions are extremely cost-effective:

- First 2 million requests per month are free
- $0.10 per million requests after free tier
- No charges for function duration
- No regional data transfer fees

Compare this to Lambda@Edge:
- Higher per-request cost
- Charged for function duration
- Regional data transfer fees apply

## Conclusion

Using CloudFront Functions for Hugo page routing provides a fast, cost-effective solution for handling URL rewrites. The simple implementation, immediate deployment, and minimal maintenance make it the ideal choice over Lambda@Edge for this specific use case.

Remember to:
- Test thoroughly before deployment
- Monitor performance metrics
- Keep the function code simple and focused
- Document any custom routing rules

## Additional Resources

- [CloudFront Functions Documentation](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-functions.html)
- [CloudFront Functions vs Lambda@Edge](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/edge-functions-compare.html)
- [Hugo Documentation](https://gohugo.io/hosting-and-deployment/hosting-on-aws/)