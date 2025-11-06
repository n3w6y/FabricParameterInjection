# Microsoft Fabric PowerBI Report Parameter Injection - Architectural Options

## Executive Summary

This document outlines architectural options for enabling parameter injection into Microsoft Fabric hosted PowerBI reports for users with view-only permissions. The goal is to provide a workflow similar to PowerBI templates (.pbit) where users receive a parameter input interface before the report loads.

**Key Challenge**: Users with view-only permissions cannot modify report parameters or data sources, and URL-based filtering can be bypassed by end users, making it unsuitable for secure parameter injection.

---

## Background & Context

### PowerBI Template Behavior
PowerBI templates (.pbit files) provide an ideal user experience:
- Users open the template and receive a parameter input dialog
- After entering parameters, the report loads with those values
- The template contains report definitions without data

### Current Limitations
1. **PBIT files cannot be published to Power BI Service/Fabric** - they must be converted to .pbix files first
2. **Parameters cannot be modified in the web service** - only in Power BI Desktop
3. **URL filtering is not secure** - users can modify URL parameters directly
4. **View-only users cannot change data source parameters** - they can only view the report as published

### Security Considerations
Recent research (Nokod Security, 2024-2025) has revealed significant data leakage vulnerabilities in Power BI:
- View-only users can access underlying semantic model data beyond what's visible in reports
- DAX injection attacks can lead to external data leakage
- URL parameters can be manipulated by end users
- These vulnerabilities underscore the importance of proper security architecture

---

## Architectural Options

### Option 1: Custom Web Application with Embedded Reports + RLS

**Architecture**: Build a custom web application that presents a parameter input form, then embeds the PowerBI report with Row-Level Security (RLS) to enforce filtering.

#### Implementation Details

1. **User Authentication**
   - Users authenticate via Microsoft Sign-In (Azure AD/Entra ID)
   - Web application uses OAuth 2.0 flow to obtain user identity

2. **Parameter Collection**
   - Custom web form presents parameter input UI
   - Parameters are validated on the server side
   - Form submission triggers embed token generation

3. **Embed Token Generation**
   - Web application backend generates embed token using Power BI REST API
   - Token includes RLS identity with parameters encoded as username
   - Example: `POST https://api.powerbi.com/v1.0/myorg/reports/{reportId}/GenerateToken`

4. **RLS Configuration**
   - PowerBI report configured with dynamic RLS roles
   - DAX expressions use `USERNAME()` or `USERPRINCIPALNAME()` functions
   - Parameters passed via the identity's username field in format: `param1|param2|param3`
   - DAX uses `PATHITEM()` function to extract individual parameters

5. **Report Embedding**
   - JavaScript embedded using PowerBI JavaScript API
   - Report loads with RLS filtering enforced server-side
   - Users cannot bypass filtering (unlike URL parameters)

#### Code Example (Conceptual)

**Backend (C#/Node.js/Python):**
```csharp
// Generate embed token with RLS identity
var identities = new List<EffectiveIdentity>
{
    new EffectiveIdentity(
        username: $"{userInput.Region}|{userInput.Department}|{userInput.Year}",
        roles: new List<string> { "ParameterizedRole" },
        datasets: new List<string> { datasetId }
    )
};

var tokenRequest = new GenerateTokenRequest(
    accessLevel: "View",
    identities: identities
);

var embedToken = await client.Reports.GenerateTokenAsync(reportId, tokenRequest);
```

**Frontend (JavaScript):**
```javascript
// Embed configuration with token
const embedConfig = {
    type: 'report',
    id: reportId,
    embedUrl: embedUrl,
    accessToken: embedToken,
    tokenType: models.TokenType.Embed,
    permissions: models.Permissions.Read,
    settings: {
        filterPaneEnabled: false,
        navContentPaneEnabled: true
    }
};

// Embed report
const report = powerbi.embed(reportContainer, embedConfig);
```

**RLS DAX in PowerBI:**
```dax
[Region] = PATHITEM(USERNAME(), 1, TEXT)
&& [Department] = PATHITEM(USERNAME(), 2, TEXT)
&& YEAR([Date]) = VALUE(PATHITEM(USERNAME(), 3, TEXT))
```

#### Pros
- ✅ **Secure**: RLS filtering cannot be bypassed by users
- ✅ **Flexible**: Complete control over parameter input UI/UX
- ✅ **View-only compatible**: Users only need view permissions on the report
- ✅ **Scalable**: Can handle complex parameter scenarios
- ✅ **Audit trail**: Application can log parameter selections
- ✅ **User authentication integrated**: Leverages existing Microsoft identity

#### Cons
- ❌ **Development effort**: Requires custom web application development
- ❌ **Infrastructure**: Needs hosting for web application and backend services
- ❌ **Service principal setup**: Requires Azure AD app registration with proper permissions
- ❌ **RLS complexity**: DAX expressions can become complex with many parameters
- ❌ **Workspace permissions**: Service principal needs Member/Admin role for embed token generation
- ❌ **Maintenance**: Application code and dependencies need ongoing maintenance

#### Required Permissions & Setup
- **Azure AD App Registration**: With delegated permissions for Power BI Service
- **Power BI Admin Portal**: Enable "Allow service principals to use Power BI APIs"
- **Workspace Access**: Service principal with Member or Admin role
- **API Permissions**: Minimum `Report.Read.All`, `Dataset.Read.All`

#### Estimated Complexity
**Medium to High** - Requires full-stack development, Azure AD integration, and Power BI API expertise.

---

### Option 2: Custom Web Application with JavaScript API Filters

**Architecture**: Build a custom web application that collects parameters, then embeds the report using JavaScript API filters applied during initialization.

#### Implementation Details

1. **User Authentication**
   - Microsoft Sign-In authentication
   - OAuth flow to obtain access token

2. **Parameter Collection**
   - Custom web form for parameter input
   - Client-side or server-side validation

3. **Embed Token Generation**
   - Standard embed token without RLS
   - Token provides view-only access

4. **Filter Application**
   - Filters constructed in JavaScript
   - Applied during embed configuration (before report loads)
   - Filters enforce parameter values on report visuals

5. **Report Embedding**
   - PowerBI JavaScript API used for embedding
   - Filters passed in `embedConfiguration.filters` array

#### Code Example (Conceptual)

**JavaScript:**
```javascript
// Construct filters from user input
const filters = [
    {
        $schema: "http://powerbi.com/product/schema#basic",
        target: {
            table: "Sales",
            column: "Region"
        },
        operator: "In",
        values: [userInput.region]
    },
    {
        $schema: "http://powerbi.com/product/schema#basic",
        target: {
            table: "Sales",
            column: "Department"
        },
        operator: "In",
        values: [userInput.department]
    },
    {
        $schema: "http://powerbi.com/product/schema#advanced",
        target: {
            table: "Calendar",
            column: "Year"
        },
        logicalOperator: "And",
        conditions: [
            {
                operator: "Is",
                value: userInput.year
            }
        ]
    }
];

// Embed configuration
const embedConfig = {
    type: 'report',
    id: reportId,
    embedUrl: embedUrl,
    accessToken: embedToken,
    tokenType: models.TokenType.Embed,
    filters: filters,  // Filters applied at initialization
    settings: {
        filterPaneEnabled: false  // Hide filter pane from users
    }
};

const report = powerbi.embed(reportContainer, embedConfig);
```

#### Pros
- ✅ **Simpler than RLS**: No need for complex DAX expressions
- ✅ **View-only compatible**: Users only need view permissions
- ✅ **Flexible UI**: Custom parameter input interface
- ✅ **Multiple filter types**: Supports basic, advanced, top-N, relative date filters
- ✅ **Filter hiding**: Can hide filter pane from users
- ✅ **Lower workspace permissions**: Doesn't require Member/Admin for basic embedding

#### Cons
- ❌ **NOT SECURE**: Users can inspect JavaScript, extract embed token, and modify filters
- ❌ **Browser manipulation**: Tech-savvy users can bypass filters via browser dev tools
- ❌ **No server-side enforcement**: Filters are client-side only
- ❌ **Security warning**: Microsoft explicitly states "This isn't a security enhancement"
- ❌ **Unsuitable for sensitive data**: Should not be used where data access control is required
- ❌ **Token exposure**: Embed tokens visible in browser can be used to access report without filters

#### Security Warning
⚠️ **This approach is NOT recommended for scenarios requiring data security.** Per Microsoft documentation: "URL filters can be modified by end-users. For security, consider Row-level security."

The same limitation applies to JavaScript API filters - they can be bypassed by users with technical knowledge.

#### Use Cases
This option may be acceptable for:
- Non-sensitive data scenarios
- User convenience (not security) filtering
- Internal trusted user environments
- Reports where all users can see all data anyway

#### Estimated Complexity
**Medium** - Requires web development and Power BI JavaScript API knowledge, but simpler than RLS implementation.

---

### Option 3: URL-Based Parameter Passing with Pre-Navigation Page

**Architecture**: Create a simple parameter selection page that constructs a filtered report URL and redirects the user.

#### Implementation Details

1. **User Authentication**
   - Users authenticate to Power BI Service directly
   - Web page can optionally verify authentication via Microsoft Sign-In

2. **Parameter Collection**
   - Simple HTML form or web page
   - Can be static HTML or minimal web application

3. **URL Construction**
   - Parameters converted to Power BI URL filter syntax
   - Format: `?filter=Table/Column eq 'Value'`
   - Multiple filters combined with "and" operator

4. **Navigation**
   - User redirected to constructed Power BI report URL
   - Report opens in Power BI Service with filters applied

#### Code Example (Conceptual)

**HTML/JavaScript:**
```html
<!DOCTYPE html>
<html>
<head>
    <title>Report Parameter Selection</title>
</head>
<body>
    <h1>Select Report Parameters</h1>
    <form id="paramForm">
        <label>Region: <input type="text" id="region" required></label><br>
        <label>Department: <input type="text" id="department" required></label><br>
        <label>Year: <input type="number" id="year" required></label><br>
        <button type="submit">View Report</button>
    </form>

    <script>
        document.getElementById('paramForm').addEventListener('submit', function(e) {
            e.preventDefault();

            const region = document.getElementById('region').value;
            const department = document.getElementById('department').value;
            const year = document.getElementById('year').value;

            // Construct filter URL
            const baseUrl = 'https://app.powerbi.com/groups/[workspace-id]/reports/[report-id]';
            const filter = `?filter=Sales/Region eq '${encodeURIComponent(region)}' ` +
                          `and Sales/Department eq '${encodeURIComponent(department)}' ` +
                          `and Calendar/Year eq ${year}`;

            // Navigate to filtered report
            window.location.href = baseUrl + filter;
        });
    </script>
</body>
</html>
```

#### URL Filter Syntax
Power BI supports query string filtering with the following syntax:

- **Basic filter**: `?filter=Table/Column eq 'Value'`
- **Multiple filters**: `and` operator to combine
- **Operators**: `eq`, `ne`, `lt`, `le`, `gt`, `ge`, `in`
- **Example**: `?filter=Store/Territory eq 'NC' and Store/Chain eq 'Fashions Direct'`

#### Limitations
- Maximum 10 filter expressions
- URL length limit of 2,000 characters (including escape codes)
- Special characters must be URL encoded

#### Pros
- ✅ **Simplest implementation**: Can be a static HTML page
- ✅ **No backend required**: Client-side only
- ✅ **No API permissions needed**: Uses native Power BI URL features
- ✅ **No service principal setup**: Works with user authentication
- ✅ **Minimal maintenance**: Static page or simple web hosting
- ✅ **Fast development**: Can be built in hours

#### Cons
- ❌ **NOT SECURE**: Users can easily modify URL parameters
- ❌ **Highly insecure**: Simple for any user to bypass by editing the URL
- ❌ **No data protection**: Cannot be used for data access control
- ❌ **Filter visibility**: Users can see and modify all filter values
- ❌ **No audit trail**: Cannot track what users actually viewed
- ❌ **Limited complexity**: 10 filter limit, 2000 character URL limit
- ❌ **Unsuitable for compliance**: Cannot meet data governance requirements

#### Security Warning
⚠️ **This is the LEAST secure option.** Microsoft explicitly states: "This isn't a security enhancement for your reports. Filters can be modified by end-users. For security, consider Row-level security."

Any user can:
- Edit the URL in the browser address bar
- Share unfiltered URLs with others
- Create bookmarks with different filter values
- Access data they shouldn't see

#### Use Cases
This option may be acceptable for:
- Public or internal non-sensitive reports
- User convenience features (not security)
- Scenarios where all users can access all data
- Prototyping and demonstrations

#### Estimated Complexity
**Low** - Can be implemented as a simple static HTML page.

---

### Option 4: Power BI App with Custom Parameter Guidance Page

**Architecture**: Publish report as a Power BI App and create a separate parameter guidance page that directs users to the app with recommended settings.

#### Implementation Details

1. **Report Configuration**
   - Report built with built-in slicers for key parameters
   - Published to a workspace
   - Packaged as a Power BI App

2. **App Publishing**
   - App published with appropriate permissions
   - Users granted access to the app
   - App can include multiple reports and navigation

3. **Parameter Guidance Page**
   - Separate web page or documentation
   - Provides instructions for parameter selection
   - Links to the Power BI App
   - May include suggested parameter values based on user role

4. **User Workflow**
   - User visits guidance page
   - User reads recommended parameter settings
   - User clicks link to Power BI App
   - User manually selects parameters using slicers in the report

#### Pros
- ✅ **No custom development**: Uses native Power BI features
- ✅ **No API integration**: Standard Power BI App publishing
- ✅ **Easy maintenance**: Managed through Power BI Service
- ✅ **User familiar interface**: Standard Power BI slicer controls
- ✅ **No infrastructure**: No custom hosting required
- ✅ **App benefits**: Can include multiple reports, navigation, branding

#### Cons
- ❌ **Manual parameter selection**: Users must set parameters themselves
- ❌ **No enforcement**: Users may ignore guidance
- ❌ **Not automated**: No automatic parameter injection
- ❌ **User error prone**: Users may select wrong values
- ❌ **No pre-filtering**: Report loads with default/all data first
- ❌ **Doesn't meet requirements**: Not similar to template parameter popup
- ❌ **Poor user experience**: Extra steps and potential confusion

#### Assessment
While this is the simplest "option," it doesn't meet the stated requirements for parameter injection similar to PowerBI templates. It's more of a workaround than a solution.

#### Use Cases
- When development resources are unavailable
- Temporary solution while building proper architecture
- Non-critical scenarios where user guidance is sufficient

#### Estimated Complexity
**Very Low** - Uses only native Power BI features.

---

### Option 5: Azure Functions + Power BI Embedded with Dynamic Reports

**Architecture**: Serverless architecture using Azure Functions to handle parameter collection, embed token generation, and report embedding.

#### Implementation Details

1. **User Authentication**
   - Microsoft Sign-In via Azure AD B2C or Entra ID
   - Can leverage Easy Auth on Azure App Service

2. **Parameter Collection**
   - Static web app or Azure Static Web Apps for UI
   - HTML form posts to Azure Function

3. **Azure Function for Token Generation**
   - HTTP triggered Azure Function
   - Receives parameters from frontend
   - Generates embed token with RLS identity
   - Returns token and embed configuration to client

4. **Report Embedding**
   - Frontend receives token and configuration
   - PowerBI JavaScript API embeds report
   - RLS enforces parameter filtering

5. **Serverless Benefits**
   - Auto-scaling based on demand
   - Pay-per-execution pricing
   - Managed infrastructure

#### Architecture Diagram (Conceptual)
```
User Browser
    ↓ (1) Authenticate
Azure AD / Entra ID
    ↓ (2) Access granted
Static Web App (Parameter Form)
    ↓ (3) Submit parameters
Azure Function (Token Generator)
    ↓ (4) Request embed token with RLS
Power BI Service API
    ↓ (5) Return embed token
Azure Function
    ↓ (6) Return token + config
Static Web App
    ↓ (7) Embed report with token
Power BI Embedded Report (with RLS filtering)
```

#### Code Example (Conceptual)

**Azure Function (C#):**
```csharp
[FunctionName("GenerateEmbedToken")]
public static async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequest req,
    ILogger log)
{
    // Parse parameters from request
    string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
    dynamic data = JsonConvert.DeserializeObject(requestBody);

    string region = data?.region;
    string department = data?.department;
    string year = data?.year;

    // Validate parameters
    if (string.IsNullOrEmpty(region) || string.IsNullOrEmpty(department))
    {
        return new BadRequestObjectResult("Missing required parameters");
    }

    // Create RLS identity
    var rls = new EffectiveIdentity(
        username: $"{region}|{department}|{year}",
        roles: new List<string> { "ParameterizedRole" },
        datasets: new List<string> { datasetId }
    );

    // Generate embed token
    var tokenRequest = new GenerateTokenRequest(
        accessLevel: "View",
        identities: new List<EffectiveIdentity> { rls }
    );

    var pbiClient = CreatePowerBIClient();
    var embedToken = await pbiClient.Reports.GenerateTokenAsync(reportId, tokenRequest);

    // Return configuration
    return new OkObjectResult(new {
        embedToken = embedToken.Token,
        embedUrl = embedUrl,
        reportId = reportId,
        expiresAt = embedToken.Expiration
    });
}
```

**Frontend (JavaScript):**
```javascript
// Submit parameters to Azure Function
const response = await fetch('https://[function-app].azurewebsites.net/api/GenerateEmbedToken', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
        region: document.getElementById('region').value,
        department: document.getElementById('department').value,
        year: document.getElementById('year').value
    })
});

const config = await response.json();

// Embed report
const embedConfiguration = {
    type: 'report',
    id: config.reportId,
    embedUrl: config.embedUrl,
    accessToken: config.embedToken,
    tokenType: models.TokenType.Embed,
    permissions: models.Permissions.Read
};

const report = powerbi.embed(reportContainer, embedConfiguration);
```

#### Pros
- ✅ **Serverless**: No server management required
- ✅ **Cost-effective**: Pay only for execution time
- ✅ **Scalable**: Auto-scales with demand
- ✅ **Secure**: RLS enforced server-side
- ✅ **Managed infrastructure**: Azure handles hosting, scaling, patching
- ✅ **Easy deployment**: Can use CI/CD with GitHub Actions or Azure DevOps
- ✅ **Azure ecosystem**: Integrates well with other Azure services

#### Cons
- ❌ **Azure dependency**: Requires Azure subscription
- ❌ **Cold start latency**: First request after idle may be slow (can be mitigated with Premium plan)
- ❌ **Debugging complexity**: Serverless debugging can be challenging
- ❌ **Service principal setup**: Still requires Azure AD app registration
- ❌ **Learning curve**: Requires understanding of Azure Functions and serverless patterns

#### Cost Considerations
- **Consumption Plan**: First 1 million executions free, then $0.20 per million
- **Static Web Apps**: Free tier available for simple sites
- **Power BI Embedded**: Separate cost based on capacity (A SKU starting ~$1/hour, can be paused)

#### Estimated Complexity
**Medium** - Requires Azure expertise and serverless architecture knowledge, but simpler than managing full web application infrastructure.

---

## Comparison Matrix

| Feature | Option 1: Custom Web App + RLS | Option 2: Custom Web App + JS Filters | Option 3: URL Parameters | Option 4: Power BI App + Guidance | Option 5: Azure Functions + RLS |
|---------|-------------------------------|---------------------------------------|--------------------------|-----------------------------------|--------------------------------|
| **Security** | ✅ High (RLS enforced) | ❌ Low (client-side) | ❌ Very Low (URL editable) | ⚠️ Medium (depends on RLS in report) | ✅ High (RLS enforced) |
| **Meets Requirements** | ✅ Yes | ⚠️ Partially | ❌ No | ❌ No | ✅ Yes |
| **Development Effort** | ⬆️ High | ⬆️ Medium | ⬇️ Very Low | ⬇️ Very Low | ⬆️ Medium |
| **Infrastructure Cost** | ⬆️ Medium-High | ⬆️ Medium-High | ⬇️ Minimal | ⬇️ None | ⬇️ Low |
| **Scalability** | ⚠️ Depends on hosting | ⚠️ Depends on hosting | ✅ Native Power BI scaling | ✅ Native Power BI scaling | ✅ Auto-scaling |
| **Maintenance** | ⬆️ Medium | ⬆️ Medium | ⬇️ Minimal | ⬇️ Minimal | ⬇️ Low |
| **User Experience** | ✅ Excellent | ✅ Excellent | ⚠️ Fair | ❌ Poor | ✅ Excellent |
| **Audit Trail** | ✅ Yes | ⚠️ Limited | ❌ No | ❌ No | ✅ Yes |
| **View-Only Compatible** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| **Data Governance** | ✅ Compliant | ❌ Not compliant | ❌ Not compliant | ⚠️ Depends | ✅ Compliant |
| **Time to Deploy** | ⬆️ 4-8 weeks | ⬆️ 2-4 weeks | ⬇️ 1-3 days | ⬇️ 1-2 days | ⬆️ 3-6 weeks |

---

## Recommendations

### Primary Recommendation: Option 5 - Azure Functions + Power BI Embedded with RLS

**Rationale:**
- **Secure**: RLS filtering enforced server-side, cannot be bypassed
- **Cost-effective**: Serverless pricing, pay only for what you use
- **Scalable**: Auto-scales based on demand without infrastructure management
- **Meets requirements**: Provides parameter popup experience before report loads
- **Modern architecture**: Aligns with cloud-native best practices
- **Lower ongoing costs**: Minimal infrastructure management overhead

**Best for:**
- Organizations already using Azure
- Projects requiring secure data filtering
- Teams with cloud/serverless expertise
- Scenarios needing cost-effective scaling

### Alternative Recommendation: Option 1 - Custom Web Application + RLS

**Rationale:**
- **Most control**: Full control over infrastructure and deployment
- **Flexible**: Can accommodate complex business logic and workflows
- **Secure**: RLS enforcement prevents filter bypass
- **Platform agnostic**: Can deploy on any hosting platform

**Best for:**
- Organizations with existing web application infrastructure
- Projects requiring complex workflow integration
- Teams preferring traditional web application architecture
- On-premises or specific hosting requirements

### NOT Recommended for Production (Security Reasons)

**Options 2, 3, and 4** are not recommended for production use where:
- Data security is required
- Regulatory compliance is needed (GDPR, HIPAA, etc.)
- Users should only see specific data
- Audit trails are necessary

These options may be suitable only for:
- Prototyping and proof-of-concept
- Non-sensitive demonstration environments
- Scenarios where all users can access all data anyway

---

## Implementation Roadmap (Option 5 - Recommended)

### Phase 1: Azure Setup & Configuration (Week 1-2)
1. Create Azure subscription (if needed)
2. Register Azure AD application
3. Configure API permissions for Power BI Service
4. Create service principal
5. Add service principal to Power BI workspace (Member/Admin role)
6. Enable service principal in Power BI admin portal

### Phase 2: Power BI Report Configuration (Week 2-3)
1. Design report data model
2. Create dynamic RLS roles with DAX expressions
3. Test RLS with sample parameters
4. Publish report to workspace
5. Document parameter schema and validation rules

### Phase 3: Azure Function Development (Week 3-4)
1. Create Azure Function App
2. Implement token generation function
   - Parameter validation
   - RLS identity creation
   - Embed token generation
   - Error handling and logging
3. Configure authentication/authorization
4. Set up Application Insights for monitoring
5. Implement unit and integration tests

### Phase 4: Frontend Development (Week 4-5)
1. Create Azure Static Web App or simple web app
2. Design parameter input form UI
3. Implement form validation
4. Integrate Microsoft Sign-In authentication
5. Implement Azure Function API calls
6. Implement Power BI JavaScript embedding
7. Add error handling and loading states

### Phase 5: Testing & Security Review (Week 5-6)
1. Functional testing
   - Parameter validation
   - RLS filtering verification
   - Different user scenarios
2. Security testing
   - Attempt filter bypass
   - Token expiration handling
   - Unauthorized access attempts
3. Performance testing
   - Load testing
   - Cold start optimization
4. User acceptance testing

### Phase 6: Deployment & Documentation (Week 6)
1. Set up production environment
2. Configure CI/CD pipeline
3. Deploy to production
4. Create user documentation
5. Create admin/maintenance documentation
6. Train support staff

---

## Cost Estimates (Option 5 - Azure Functions)

### Azure Services (Monthly Estimates)

**Assuming moderate usage: 10,000 report views/month**

| Service | Tier | Estimated Cost |
|---------|------|----------------|
| Azure Functions | Consumption Plan | ~$0-20/month (likely free tier) |
| Azure Static Web Apps | Standard | ~$9/month (Free tier available) |
| Application Insights | Pay-as-you-go | ~$5-15/month |
| Azure AD | Free tier | $0 |
| **Azure Total** | | **~$15-45/month** |

**Power BI Licensing:**
| License Type | Cost | Notes |
|--------------|------|-------|
| Power BI Embedded (A SKU) | ~$1/hour (~$730/month if always on) | Can be paused when not in use |
| Power BI Embedded (A SKU) | ~$50-200/month with pause strategy | Paused during off-hours |
| Power BI Premium Per User | $20/user/month | Alternative licensing model |

**Total Estimated Monthly Cost: $65-245/month** (depending on Power BI licensing strategy and usage)

### Cost Optimization Strategies
1. **Pause Power BI Embedded capacity** during off-hours (nights/weekends)
2. **Use free tier** for Azure Static Web Apps initially
3. **Implement caching** to reduce Azure Function executions
4. **Use Premium Per User** licensing if user count is low and predictable

---

## Security Best Practices

### 1. Token Management
- Embed tokens should have short expiration times (1-2 hours maximum)
- Implement token refresh mechanism for long-running sessions
- Never expose tokens in client-side code or logs
- Store tokens only in memory, not in browser storage

### 2. Parameter Validation
- Validate all parameters server-side (never trust client input)
- Implement whitelist validation for allowed parameter values
- Sanitize inputs to prevent injection attacks
- Log all parameter selections for audit purposes

### 3. RLS Implementation
- Design RLS roles with principle of least privilege
- Test RLS thoroughly with different user scenarios
- Never rely on obfuscation or hiding UI elements for security
- Document RLS logic clearly for maintenance

### 4. Authentication & Authorization
- Always use Microsoft Sign-In (Azure AD) for authentication
- Implement proper authorization checks before token generation
- Use Azure AD security groups for role-based access if needed
- Enable MFA for administrative accounts

### 5. Network Security
- Use HTTPS for all communications
- Enable CORS restrictions on Azure Functions
- Implement rate limiting to prevent abuse
- Use Azure Front Door or Application Gateway for additional protection

### 6. Monitoring & Alerting
- Log all authentication attempts
- Monitor for unusual parameter patterns
- Set up alerts for failed token generation attempts
- Regularly review Application Insights logs

### 7. Data Governance
- Classify data sensitivity levels
- Implement data retention policies
- Document who has access to what data
- Regular access reviews and audits

---

## Technical Considerations

### Browser Compatibility
- Power BI JavaScript API supports: Chrome, Edge, Firefox, Safari
- Test embed experience across all target browsers
- Consider mobile browser support if needed

### Performance Optimization
- Implement report performance best practices (DAX optimization, data model design)
- Use report bookmarks for common parameter combinations
- Consider pre-aggregating data for common filter scenarios
- Monitor report load times and optimize slow visuals

### Accessibility
- Ensure parameter input forms are screen reader accessible
- Power BI embedded reports support keyboard navigation
- Follow WCAG 2.1 AA guidelines
- Test with assistive technologies

### Internationalization
- Consider multi-language support for parameter labels
- Power BI supports multiple languages in reports
- Date/number formatting based on user locale
- Currency conversion if needed

---

## Risk Assessment

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| **Users bypass filtering** | High (Options 2-3) / Low (Options 1,5) | Critical | Use RLS enforcement (Options 1 or 5) |
| **Embed token exposure** | Medium | High | Short expiration times, HTTPS only, no client logging |
| **RLS misconfiguration** | Medium | Critical | Thorough testing, code review, security audit |
| **Service principal compromise** | Low | Critical | Secure credential storage, regular rotation, minimal permissions |
| **Power BI service outage** | Low | High | Monitor status, communicate with users, plan maintenance windows |
| **Azure Function cold start** | Medium | Low | Consider Premium plan, implement health check warming |
| **Scaling issues** | Low | Medium | Load testing, monitor metrics, auto-scaling configuration |
| **Compliance violations** | Low | Critical | Data classification, access reviews, audit logging |

---

## Alternative Technologies Considered

### Power Apps Portals
- Could provide parameter input interface
- Limited Power BI embedding capabilities
- Higher licensing costs
- Not recommended for this use case

### SharePoint + Power BI Web Part
- Easier integration for SharePoint users
- Still faces URL/filter bypass security issues
- Doesn't provide true parameter injection
- Not recommended as primary solution

### Power Automate + Power Apps
- Could orchestrate parameter collection and report access
- Complex architecture for this use case
- Performance concerns
- Overkill for parameter injection needs

### Custom React/Angular SPA
- More control over frontend
- Higher development and maintenance overhead
- Can be combined with Options 1 or 5
- Consider only if advanced UI features needed

---

## Conclusion

For Microsoft Fabric hosted PowerBI reports with view-only permissions, implementing parameter injection similar to PowerBI templates requires a custom solution. The recommended approach is **Option 5: Azure Functions + Power BI Embedded with RLS**, which provides:

1. **Security**: Server-side RLS enforcement prevents filter bypass
2. **User Experience**: Custom parameter form before report loads
3. **Scalability**: Serverless architecture scales automatically
4. **Cost-Effectiveness**: Pay-per-use pricing model
5. **Maintainability**: Managed Azure services reduce operational overhead

The key technical requirements are:
- Azure AD app registration with Power BI API permissions
- Service principal with workspace Member/Admin permissions
- Dynamic RLS configuration in Power BI reports
- Secure embed token generation with RLS identities
- Custom web frontend for parameter collection

**Critical Security Note**: URL-based filtering and client-side JavaScript filters (Options 2-3) are NOT secure and should never be used for data access control. Only server-side RLS enforcement (Options 1 or 5) provides the necessary security for parameter-based data filtering.

---

## Next Steps

1. **Stakeholder Review**: Present architectural options to stakeholders
2. **Budget Approval**: Confirm Azure subscription and licensing budget
3. **Team Assignment**: Identify developers for implementation
4. **Timeline Planning**: Create detailed project timeline based on roadmap
5. **Proof of Concept**: Build small POC with sample report to validate approach
6. **Security Review**: Engage security team for architecture review
7. **Procurement**: Set up Azure subscription and Power BI licensing if needed

---

## References & Resources

### Microsoft Documentation
- [Power BI Embedded Analytics](https://learn.microsoft.com/en-us/power-bi/developer/embedded/embedded-analytics-power-bi)
- [Row-Level Security (RLS) Guidance](https://learn.microsoft.com/en-us/power-bi/guidance/rls-guidance)
- [Generate Embed Token API](https://learn.microsoft.com/en-us/rest/api/power-bi/embed-token)
- [Power BI JavaScript API](https://github.com/microsoft/PowerBI-JavaScript)
- [Azure Functions Documentation](https://learn.microsoft.com/en-us/azure/azure-functions/)
- [URL Filter Parameters](https://learn.microsoft.com/en-us/power-bi/collaborate-share/service-url-filters)

### Community Resources
- [Power BI Embedded Playground](https://playground.powerbi.com/)
- [Power BI Community Forums](https://community.fabric.microsoft.com/)
- [Power BI Developer Samples](https://github.com/microsoft/PowerBI-Developer-Samples)

### Security Research
- [Nokod Security - Power BI Data Leakage Vulnerability (2024)](https://nokodsecurity.com/blog/in-plain-sight-how-microsoft-power-bi-reports-expose-sensitive-data-on-the-web/)
- [Power BI Analyzer Tool](https://github.com/Nokod/PBAnalyzer)

---

**Document Version**: 1.0
**Last Updated**: 2025-11-06
**Author**: Architecture Analysis for Fabric Parameter Injection
**Status**: Draft for Review
