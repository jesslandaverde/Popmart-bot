import asyncio
from playwright.async_api import async_playwright
import time

PRODUCT_URL = "https://m.popmart.com/us/pop-now/set/195"

async def run_bot():
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)  # headless=True is required for cloud/VPS
        context = await browser.new_context(
            viewport={'width': 390, 'height': 844},
            user_agent="Mozilla/5.0 (iPhone; CPU iPhone OS 14_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/14.0 Mobile/15E148 Safari/604.1"
        )
        page = await context.new_page()

        # If login is needed, run this once locally to manually log in and export cookies
        await page.goto("https://m.popmart.com/us/user/login")
        print("👉 Log in manually, then press ENTER to continue...")
        input()

        await context.storage_state(path="session.json")

        while True:
            try:
                print("🔍 Checking product availability...")
                await page.goto(PRODUCT_URL)
                await page.wait_for_load_state("networkidle")

                # Try to find "Add to Cart" button
                add_button = await page.query_selector('//button[contains(text(), "Add to Cart")]')

                if add_button:
                    print("✅ In stock! Adding to cart...")
                    await add_button.click()
                    await asyncio.sleep(2)

                    # Proceed to cart
                    await page.goto("https://m.popmart.com/us/cart")
                    await page.wait_for_load_state("networkidle")

                    checkout_btn = await page.query_selector('//button[contains(text(), "Checkout")]')

                    if checkout_btn:
                        print("🚀 Checkout button found! Attempting checkout...")
                        await checkout_btn.click()
                        print("✅ Checkout started — complete payment manually if needed.")
                        break  # Stop loop after successful start
                    else:
                        print("❌ Checkout button not found. Retrying...")
                        await asyncio.sleep(5)
                        continue
                else:
                    print("⏳ Item not in stock. Retrying...")
            except Exception as e:
                print(f"⚠️ Error: {e}. Retrying in 10 seconds...")

            await asyncio.sleep(10)

        await browser.close()

asyncio.run(run_bot())
