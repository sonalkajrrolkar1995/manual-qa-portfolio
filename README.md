# Manual QA Portfolio

I put this together to show how I actually work through a testing engagement - not just running test cases, but understanding what can go wrong and why it matters.

I tested three applications: SauceDemo (a web e-commerce demo), ReqRes (a mock REST API), and Restful Booker (a booking REST API). Everything here comes from real testing sessions - the risks I identified, the bugs I found, the edge cases I probed, and the thinking behind it all.

This is not a list of test cases. It is how I approach QA in a production environment.

---

| Document | What is in it |
|---|---|
| [01 - Product Understanding](01-product-understanding.md) | How each system works and what the critical flows are |
| [02 - Risk Analysis](02-risk-analysis.md) | What can fail and why it matters - functional, data, business, security, UX |
| [03 - Test Design](03-test-design.md) | The scenarios I would run - positive, negative, boundary, edge cases |
| [04 - Exploratory Testing](04-exploratory-testing.md) | Four structured sessions - what I set out to test, what I found, what it means |
| [05 - Defect Reports](05-defect-reports.md) | 13 bugs documented the way I would log them in Jira |
| [06 - API Analysis](06-api-analysis.md) | How the APIs behave, where the validation is missing, what breaks |
| [07 - System Thinking](07-system-thinking.md) | The bigger picture - architecture risks, data flow, integration and reliability |

---

Applications I tested:

- SauceDemo - https://www.saucedemo.com
- ReqRes - https://reqres.in
- Restful Booker - https://restful-booker.herokuapp.com
