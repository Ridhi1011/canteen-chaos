# Debug Log

I worked through the 10 reported bugs and also found three more issues while testing the application. I fixed those as well and added a small test file for some of the bugs.

For bugs that depended on the time, I ran the server with a timezone that put the canteen within its opening hours. I reset the backend data between test runs using `git checkout -- backend/data/`.

---

## CC-01 — Search suggestions appear behind the category bar

**Reproduced:** I typed `ch` in the search box. The suggestions appeared, but the category bar covered part of the list, making some suggestions difficult to click.

**Cause:** In `frontend/style.css`, `.search-wrap` had a `z-index` of `1`, while `.cat-tabs` had a `z-index` of `40`. The suggestions had a higher `z-index`, but they were still affected by the stacking order of their parent.

**Fix:** I changed the `z-index` of `.search-wrap` from `1` to `41`. This brought the search area above the category bar.

**Checked:** I checked that the suggestions appear above the category bar and can be clicked. The category bar still works normally.

**Time:** 10-15min

---

## CC-02 — Dish names and prices are hard to read in dark mode

**Reproduced:** I switched to dark mode and noticed that the dish names and prices were almost invisible. I also cleared `localStorage`, enabled dark mode on my system, and reloaded the page. The same issue appeared on the first load.

**Cause:** The `.dish-body` class in `frontend/style.css` used the fixed text colour `#2b2118`. This colour was too dark for the dark theme. The dish names and prices inherited this colour, while the smaller text remained readable because it used a theme variable.

**Fix:** I changed the text colour to `var(--ink)`, which adjusts according to the selected theme. I initially tried an orange colour, but that also affected the light theme, so I changed it to the theme variable instead.

**Checked:** I checked the text in both light and dark modes, switched between the themes, and tested the initial page load with system dark mode enabled.

**Time:** 5min

---

## CC-03 — The menu overflows on smaller screens

**Reproduced:** I checked the menu in Chrome DevTools at phone widths, including 375px and around 600–700px. The page scrolled sideways, and some Add to Cart buttons were cut off.

**Cause:** Two CSS rules were causing the problem. The dish images had `width: auto`, which allowed them to become wider than their grid columns. The dish name buttons also used `width: max-content`, preventing long names from wrapping. As a result, the cards could become wider than the available space.

**Fix:** In the mobile media query in `frontend/style.css`, I changed the image width to `100%`, removed `width: max-content` from the dish name button, and changed the grid columns to `minmax(0, 1fr)` so they could shrink properly. I also removed my earlier grid-width changes because they did not solve the actual cause.

**Checked:** I tested at widths of 360px, 375px, 414px, and 700px. The page no longer scrolls sideways, the buttons are visible, and long dish names wrap onto multiple lines. The desktop layout remained unchanged.

**Time:** 10min

---

## CC-04 — Add to Cart and favourite buttons do not work on tablets

**Reproduced:** At a width of 800px, clicking Add to Cart or the favourite star did nothing. Both worked at 375px and 1024px. When I inspected the element under the pointer at 800px, it was a pseudo-element on the card rather than the button.

**Cause:** A media query for widths between 761px and 900px added two invisible `::after` overlays. One covered the bottom of the dish card, where the Add to Cart button was located. The other covered the top-right corner, where the favourite star appeared. These overlays were catching the clicks.

**Fix:** I added `pointer-events: none` to both pseudo-elements. This allows clicks to reach the buttons underneath while keeping the overlays used for unavailable dishes.

**Checked:** I tested both buttons at 800px and 900px. They worked, and the layout at phone and laptop widths remained unchanged.

**Time:** 15min

---

## CC-05 — The category bar scrolls away on phones

**Reproduced:** At phone widths below 480px, the search and category bars disappeared when I scrolled down the menu. On wider screens, the filter bar stayed at the top as expected.

**Cause:** The filter bar used `position: sticky`, but the mobile CSS also set `overflow-x: hidden` on `.view`. This changed the scrolling behaviour of the parent, causing the sticky bar to stick within that container instead of behaving as expected during page scrolling.

**Fix:** I removed `overflow-x: hidden` from `.view`. After fixing the menu overflow in CC-03, this rule was no longer needed.

**Checked:** I scrolled through the menu at phone width and confirmed that the filter bar stays at the top. I also checked that the page still does not scroll sideways.

**Time:** 15min

---

## CC-06 — Customers can order more than the available stock

**Reproduced:** I set the Samosa stock to 2 on the Counter page, added 5 samosas to the cart, and placed the order. The order went through, and the stock in `menu.json` became `-3`.

**Cause:** I found two problems in `backend/logic/validation.js`. The `validateLine()` function checked whether a dish was sold out, but it did not compare the requested quantity with the available stock. The `reserveStock()` function also deducted the quantity without checking whether enough stock remained.

**Fix:** I added a stock check in `validateLine()` so an order is rejected when the requested quantity exceeds the available stock. I also added a check in `reserveStock()` before deducting stock, so the stock cannot become negative if another order uses the remaining stock first.

**Checked:** I tested an order for 5 samosas when only 2 were available. The order was rejected with an appropriate stock message. Ordering exactly 2 worked and reduced the stock to 0. Normal orders continued to work.

**Time:** 20min

---

## CC-07 — Cancelling an order reduces stock again

**Reproduced:** I started with 10 items in stock and placed an order for 2. The stock dropped to 8. After cancelling the order, it dropped to 6 instead of returning to 10.

**Cause:** The `releaseStock()` function in `backend/logic/validation.js` was subtracting the cancelled quantity from the stock. Cancellation should return the items to stock, not deduct them again.

**Fix:** I changed the subtraction to addition in `releaseStock()`, so cancelling an order restores the quantity.

**Checked:** I placed an order for 2 items, confirmed that the stock changed from 10 to 8, and cancelled it. The stock returned to 10. I also checked repeated order and cancellation cycles.

**Time:** 20min

---

## CC-08 — An expired coupon still works

**Reproduced:** The FRESHERS24 coupon was not visible in the coupon list, but entering it manually with a cart value above ₹100 still applied a ₹30 discount. Its expiry date in `coupons.json` was `2024-09-30`.

**Cause:** The `GET /api/coupons` route filtered out expired coupons, so the coupon was hidden from the list. However, `applyCoupon()` in `backend/logic/pricing.js` did not check the expiry date when validating a coupon for a cart quote or order.

**Fix:** I added an expiry-date check inside `applyCoupon()`. Since both the quote and order flow use this function, the validation now covers both cases.

**Checked:** FRESHERS24 was rejected with an expiry message, and an order using it could not be placed. I also checked that BYTE10 and other valid coupons continued to work.

**Time:** 15-20min

---

## CC-09 — The menu displays too many dishes at once

**Reproduced:** The frontend requested 5 dishes per page, but the menu displayed all 37 dishes immediately. The count also showed that the requested page size was 5, even though every dish was visible.

**Cause:** The `paginate()` function in `backend/logic/search.js` calculated the correct page using `slice()`, but returned the full list in `items` instead of the sliced result. The page count and `hasMore` values were already being calculated correctly.

**Fix:** I changed the function to return the sliced list of items rather than the complete list.

**Checked:** The first page displayed 5 dishes. Using Load more added another 5 at a time, and loading stopped when there were no more dishes.

**Time:** 20-25min

---

## CC-10 — Sorting by price works in the wrong order

**Reproduced:** When I selected Price: low to high, the most expensive dishes appeared first. Selecting high to low showed the cheapest dishes first.

**Cause:** The price comparators in the `SORTERS` object in `backend/logic/search.js` were reversed. The ascending option used `b.price - a.price`, while the descending option used `a.price - b.price`.

**Fix:** I corrected both comparators. The ascending option now uses `a.price - b.price`, and the descending option uses `b.price - a.price`.

**Checked:** I checked that low to high starts with the cheapest dish and high to low starts with the most expensive. I also tested sorting while a search term was entered.

**Time:** 15min

---

# Extra Credit

## Extra 1 — My Orders page is always empty

**How I noticed:** While testing order cancellations, I opened My Orders and noticed that it always showed no orders, even though the browser had saved order tokens.

**Reproduced:** I placed an order and opened My Orders. The page remained empty.

**Cause:** In `backend/routes/orders.routes.js`, the route checked for `req.query.memberId` but filtered orders using `req.query.userId`. Since the page sent `memberId` and not `userId`, the filter did not match the user's orders.

**Fix:** I changed the filter to use `req.query.memberId`, matching the parameter checked by the route.

**Checked:** My Orders displayed my orders, and I could cancel an order from that page. The Counter page continued to display all orders because it does not send `memberId`.


---

## Extra 2 — Orders get stuck at ready and cannot be rated

**How I noticed:** I ran the existing `smoke.js` test against the updated server. Five checks failed, including the transition from ready to picked and the rating checks.

**Reproduced:** An order could reach the ready status, but it could not move to picked. Since rating required the order to be picked, the rating flow also failed.

**Cause:** In `backend/logic/status.js`, the `TERMINAL` list included `ready`. The application treated ready as a final status, so no further transition was allowed.

**Fix:** I removed `ready` from `TERMINAL` so the order could move to picked.

**Checked:** The smoke test went from 85/90 passing checks to 90/90. A ready order could be marked as picked, after which the student could rate it.


---

## Extra 3 — Coupon uses are not deducted correctly

**How I noticed:** While reading the order route during the CC-07 investigation, I noticed that the code for using a coupon was not reducing its remaining uses.

**Reproduced:** The coupon's `usesLeft` value stayed the same after an order used it. Cancelling the order then added one use back, allowing the remaining uses to increase incorrectly.

**Cause:** The order-placement code assigned `usesLeft: c.usesLeft`, which kept the original value instead of decreasing it. The cancellation code already added one use back.

**Fix:** I changed the order-placement code to use `c.usesLeft - 1`, so placing an order deducts one use and cancelling it restores one use.

**Checked:** I checked that BYTE10 decreased by one after an order used it and increased by one when that order was cancelled.

**Time:** [Enter your actual approximate time]

---

# Things I Have Not Fully Fixed or Tested

- **Refunds:** The code appears to support a transition from picked to refunded, but `allowedNext()` does not allow that transition. I left this unchanged because allowing the transition could also affect how My Orders identifies active orders. The refund flow needs a clearer decision about how refunded orders should appear.

- **Ready orders:** Removing `ready` from `TERMINAL` means an order that remains ready and uncollected for more than five minutes can appear as stalled or active on the Counter page. I left this behaviour in place because staff may need to notice orders that have not been collected.

- **Cart stock limit:** The cart still allows a customer to add more items than are currently available and displays a warning. The backend rejects the order, so invalid orders cannot be placed. I did not change the frontend cart behaviour as part of CC-06.
