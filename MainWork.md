
> **Indresh** - _(11/12/2023 05:30:41)_
```
async def get_computed_html(url): browser = await launch() page = await browser.newPage() await page.goto(url) # Wait for a moment to ensure that dynamic content is loaded await page.waitForTimeout(2000) # Get the content of the page after JavaScript has been executed content = await page.content() await browser.close() return content
```
---
