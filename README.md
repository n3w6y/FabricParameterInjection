# Fabric Parameter Injection - Project Overview

## Purpose

This project explores architectural options for enabling parameter injection into Microsoft Fabric hosted PowerBI reports for users with view-only permissions, providing a workflow similar to PowerBI templates where users can input parameters before the report loads.

## Quick Summary

### The Challenge
- PowerBI templates (.pbit) provide excellent parameter input experience but cannot be published to Power BI Service/Fabric
- Users with view-only permissions cannot modify report parameters
- URL-based filtering can be bypassed and is not secure
- Need a solution that works with Microsoft Fabric/Power BI Service for view-only users

### Recommended Solution
**Azure Functions + Power BI Embedded with Row-Level Security (RLS)**

This serverless architecture provides:
- **Secure parameter enforcement** via server-side RLS (cannot be bypassed by users)
- **Custom parameter input form** presented before report loads
- **Cost-effective scaling** with pay-per-use pricing
- **View-only compatibility** - users only need read permissions
- **Professional user experience** similar to PowerBI template workflow

### Estimated Implementation
- **Timeline**: 6 weeks
- **Cost**: ~$65-245/month (depending on Power BI licensing and usage)
- **Complexity**: Medium (requires Azure and Power BI API expertise)

## Documentation

### Main Document
📄 **[ARCHITECTURAL_OPTIONS.md](./ARCHITECTURAL_OPTIONS.md)** - Comprehensive analysis of all architectural options

This document includes:
- 5 detailed architectural options with implementation details
- Security analysis and best practices
- Complete comparison matrix
- Implementation roadmap
- Cost estimates
- Risk assessment
- Code examples

## Options Summary

| Option | Security | Complexity | Recommended |
|--------|----------|------------|-------------|
| 1. Custom Web App + RLS | ✅ High | High | ✅ Alternative |
| 2. Custom Web App + JS Filters | ❌ Low | Medium | ❌ Not Secure |
| 3. URL Parameters | ❌ Very Low | Very Low | ❌ Not Secure |
| 4. Power BI App + Guidance | ⚠️ Medium | Very Low | ❌ Doesn't Meet Requirements |
| 5. Azure Functions + RLS | ✅ High | Medium | ✅ **PRIMARY RECOMMENDATION** |

## Key Security Findings

⚠️ **Important Security Notes:**

1. **URL-based filtering is NOT secure** - Users can easily modify URL parameters in the browser
2. **JavaScript API filters are NOT secure** - Can be bypassed via browser developer tools
3. **Only Row-Level Security (RLS) provides true data protection** - Enforced server-side in Power BI
4. **Recent vulnerabilities** - View-only users can access underlying semantic model data beyond what's shown in reports (Nokod Security research, 2024-2025)

**For any scenario requiring data security or compliance, use Options 1 or 5 with RLS enforcement.**

## Technology Stack (Recommended Solution)

### Frontend
- Azure Static Web Apps (or similar static hosting)
- HTML/JavaScript for parameter input form
- Power BI JavaScript API for report embedding
- Microsoft Sign-In (Azure AD) for authentication

### Backend
- Azure Functions (Serverless)
- C# or Node.js for token generation
- Power BI REST API integration
- Application Insights for monitoring

### Power BI
- Microsoft Fabric workspace
- Power BI report with dynamic RLS roles
- DAX expressions for parameter filtering
- Service principal with workspace Member/Admin permissions

## Getting Started

### Prerequisites
- Azure subscription
- Microsoft Fabric/Power BI workspace
- Power BI Embedded capacity or Premium Per User licenses
- Azure AD tenant for authentication

### Implementation Phases

1. **Azure Setup** (Weeks 1-2)
   - Azure AD app registration
   - Service principal configuration
   - API permissions setup

2. **Power BI Configuration** (Weeks 2-3)
   - Report development
   - RLS role creation
   - Testing and validation

3. **Azure Function Development** (Weeks 3-4)
   - Token generation function
   - Parameter validation
   - Error handling

4. **Frontend Development** (Weeks 4-5)
   - Parameter input UI
   - Microsoft Sign-In integration
   - Report embedding

5. **Testing & Security Review** (Weeks 5-6)
   - Functional testing
   - Security testing
   - Performance optimization

6. **Deployment** (Week 6)
   - Production deployment
   - Documentation
   - User training

## Architecture Diagram (Recommended Solution)

```
┌─────────────┐
│ User Browser│
└──────┬──────┘
       │ (1) Authenticate via Microsoft Sign-In
       ↓
┌──────────────────┐
│ Azure AD / Entra │
└──────┬───────────┘
       │ (2) Authentication token
       ↓
┌────────────────────────┐
│ Static Web App         │
│ (Parameter Input Form) │
└──────┬─────────────────┘
       │ (3) Submit parameters
       ↓
┌────────────────────────┐
│ Azure Function         │
│ (Token Generator)      │
└──────┬─────────────────┘
       │ (4) Request embed token with RLS
       ↓
┌────────────────────────┐
│ Power BI Service API   │
└──────┬─────────────────┘
       │ (5) Return embed token
       ↓
┌────────────────────────┐
│ Azure Function         │
└──────┬─────────────────┘
       │ (6) Return token + config
       ↓
┌────────────────────────┐
│ Static Web App         │
└──────┬─────────────────┘
       │ (7) Embed report with token
       ↓
┌────────────────────────────────┐
│ Power BI Embedded Report       │
│ (with RLS filtering applied)   │
└────────────────────────────────┘
```

## Cost Breakdown (Monthly Estimates)

### Azure Services
- Azure Functions (Consumption): $0-20
- Static Web Apps: $0-9
- Application Insights: $5-15
- **Azure Total: ~$15-45**

### Power BI Licensing
- Power BI Embedded (A1 SKU with pause strategy): ~$50-200
- *OR*
- Power BI Premium Per User: $20/user

### Total Estimated Cost
**$65-245/month** (depending on licensing model and usage patterns)

## Next Steps

1. ✅ Review the comprehensive [ARCHITECTURAL_OPTIONS.md](./ARCHITECTURAL_OPTIONS.md) document
2. ⬜ Present options to stakeholders
3. ⬜ Confirm budget and Azure subscription availability
4. ⬜ Assign development team
5. ⬜ Build proof of concept
6. ⬜ Conduct security review
7. ⬜ Begin implementation following the roadmap

## Resources

### Microsoft Documentation
- [Power BI Embedded Analytics](https://learn.microsoft.com/en-us/power-bi/developer/embedded/embedded-analytics-power-bi)
- [Row-Level Security (RLS)](https://learn.microsoft.com/en-us/power-bi/guidance/rls-guidance)
- [Power BI REST API](https://learn.microsoft.com/en-us/rest/api/power-bi/)
- [Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/)

### Developer Tools
- [Power BI Embedded Playground](https://playground.powerbi.com/)
- [Power BI JavaScript API](https://github.com/microsoft/PowerBI-JavaScript)
- [Power BI Developer Samples](https://github.com/microsoft/PowerBI-Developer-Samples)

## Project Status

**Status**: Architecture Analysis Complete ✅
**Last Updated**: 2025-11-06
**Next Phase**: Stakeholder Review & Decision

## Questions or Feedback?

For questions about this architecture analysis or to discuss implementation details, please review the comprehensive documentation in [ARCHITECTURAL_OPTIONS.md](./ARCHITECTURAL_OPTIONS.md).

---

**Important Notice**: This is an architectural analysis document. Implementation should only proceed after stakeholder review, security approval, and budget confirmation.
