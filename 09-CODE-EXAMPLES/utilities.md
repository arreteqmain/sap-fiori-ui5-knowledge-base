# Utilities: formatters, validators, helpers, and reuse

## Scope and prerequisites

These examples target **UI5 1.120** public APIs, using a maintained patch or a compatible newer runtime. They use modules available in **OpenUI5 and SAPUI5**. No SAPUI5-only formatter library, external decimal package, or global application singleton is required. The exact-arithmetic helper additionally requires native `BigInt` in the supported browser; do not copy it into a runtime that does not support that JavaScript feature.

Use namespace `com.acme.orders`, mapped to `webapp`. Modules shown here are copied to the specified application paths, not executed directly as Node scripts. The examples assume ordinary UI5 module loading and, for the XML integration, the local host pattern in [views.md](./views.md#minimal-hosting-integration). The shared `OrderSet` contract uses opaque string keys, decimal-string amounts, and ISO calendar-date strings. Amount scale is explicitly two for this demonstration, not a claim that all currencies have two minor units.

[controllers.md](./controllers.md) describes lifecycle ownership, [models.md](./models.md) describes service representations and resource bundles, and [services.md](./services.md) describes safe transport errors. These paths match the repository [README](../README.md#repository-structure). Helpers here do not replace backend authorization, service validation, or authoritative accounting rules.

## Decide whether a formatter, type, or validator is needed

A formatter transforms model values into display values. It is appropriate for status labels, short descriptions, or a calendar-date display. It should be deterministic for its inputs, tolerate temporarily missing binding values, and avoid changing model state. A formatter may run many times during binding updates; network calls, MessageBoxes, and writes inside it cause unpredictable behavior.

A binding type handles formatting, parsing, and validation. For an editable amount, use a type rather than a formatter that converts a decimal into a string with no reverse conversion. UI5 can then reject invalid input without pretending the model contains a valid value. The model may retain the previous value while the control displays invalid text, so Save logic must also account for validation state.

A validator enforces a domain rule on an already defined representation. It should say whether it accepts display text or canonical model values. A regular expression for dot-separated decimals is not a locale-aware parser: German input `"1.234,50"` and English input `"1,234.50"` need locale-aware interpretation before canonical validation.

Keep pure helpers independent of controls and controllers. Passing an entire controller to a utility so it can look up three fields creates hidden dependencies and makes reuse fragile. Prefer explicit arguments and small return objects. Where a helper owns a timer or event listener, expose cancellation and make ownership clear.

## Canonical validators and payload projection

Place this complete module at `webapp/util/orderValues.js`. It accepts a strict canonical representation: six ASCII digits for the illustrative key, a trimmed customer name, and a nonnegative decimal string with exactly two fractional digits. The maximum thirteen integer digits corresponds to precision fifteen and scale two. These are application rules; adjust them only together with the service metadata and documented business contract.

```javascript
sap.ui.define([], function () {
  "use strict";

  function validOrderId(value) {
    return typeof value === "string" && /^\d{6}$/.test(value);
  }

  function validAmount(value) {
    return typeof value === "string" &&
      /^(0|[1-9]\d{0,12})\.\d{2}$/.test(value);
  }

  function validateDraft(draft) {
    const errors = {};
    const source = draft && typeof draft === "object" ? draft : {};
    if (typeof source.CustomerName !== "string" ||
        !source.CustomerName.trim() ||
        source.CustomerName.trim().length > 80) {
      errors.CustomerName = "invalidCustomer";
    }
    if (!validAmount(source.Amount)) {
      errors.Amount = "invalidAmount";
    }
    if (typeof source.Currency !== "string" ||
        !/^[A-Z]{3}$/.test(source.Currency)) {
      errors.Currency = "invalidCurrency";
    }
    return errors;
  }

  function toCreatePayload(id, draft) {
    const errors = validateDraft(draft);
    if (!validOrderId(id) || Object.keys(errors).length) {
      throw new Error("INVALID_ORDER_DRAFT");
    }
    return {
      OrderID: id,
      CustomerName: draft.CustomerName.trim(),
      Amount: draft.Amount,
      Currency: draft.Currency,
      Status: "NEW"
    };
  }

  return {
    validOrderId: validOrderId,
    validAmount: validAmount,
    validateDraft: validateDraft,
    toCreatePayload: toCreatePayload
  };
});
```

The validator returns resource keys instead of English prose or backend error text. The caller resolves those keys through `i18n`. Add `invalidAmount` and `invalidCurrency` to every required resource bundle when using this module. Validation remains useful even if a button is disabled: a handler can be invoked programmatically, another field may change, and the backend must independently enforce the same business constraints.

The projection whitelists writable fields. Do not send an entire row object that may include `__metadata`, navigation expansions, derived totals, or read-only approval fields. Rejecting an invalid key is different from changing it. `"000042"` stays `"000042"`; there is no numeric conversion, trimming of meaningful characters, or automatic left-padding of arbitrary input.

The canonical validator deliberately rejects `"01.00"` as a noncanonical amount, but key validation deliberately accepts leading zeros. Those are different domains. A generic “remove leading zeros” utility would be wrong for at least one of them. Similarly, a three-letter currency shape does not prove that a code is supported by your company; validate allowed codes against controlled domain data.

## Locale-aware decimal editing

The built-in OData Decimal type uses string model values for exact decimal representation. This complete view can be hosted by changing the bootstrap view name to `com.acme.orders.view.Amount`. It pairs with the complete controller below and does not depend on the larger Orders controller.

```xml
<!-- webapp/view/Amount.view.xml -->
<mvc:View xmlns:mvc="sap.ui.core.mvc" xmlns="sap.m"
  controllerName="com.acme.orders.controller.Amount">
  <Page title="Amount entry">
    <content>
      <VBox class="sapUiSmallMargin">
        <Label text="Amount in EUR" labelFor="amount"/>
        <Input id="amount"
          value="{
            path: 'draft>/Amount',
            type: 'sap.ui.model.odata.type.Decimal',
            formatOptions: {
              minFractionDigits: 2,
              maxFractionDigits: 2
            },
            constraints: {
              precision: 15,
              scale: 2,
              nullable: false,
              minimum: '0'
            }
          }"/>
        <Button text="Validate" press=".onValidate"/>
      </VBox>
    </content>
  </Page>
</mvc:View>
```

```javascript
// webapp/controller/Amount.controller.js
sap.ui.define([
  "sap/ui/core/mvc/Controller",
  "sap/ui/model/json/JSONModel",
  "sap/m/MessageBox"
], function (Controller, JSONModel, MessageBox) {
  "use strict";
  return Controller.extend("com.acme.orders.controller.Amount", {
    onInit: function () {
      this._draft = new JSONModel({ Amount: "1250.00" });
      this.getView().setModel(this._draft, "draft");
    },
    onValidate: function () {
      const input = this.byId("amount");
      const type = input.getBinding("value").getType();
      try {
        const canonical = type.parseValue(input.getValue(), "string");
        type.validateValue(canonical);
        this._draft.setProperty("/Amount", canonical);
        input.setValueState("None");
        input.setValueStateText("");
        MessageBox.success("The amount is valid.");
      } catch (error) {
        input.setValueState("Error");
        input.setValueStateText("Enter a valid nonnegative amount.");
      }
    },
    onExit: function () {
      this._draft.destroy();
    }
  });
});
```

Explicitly parsing the control's current text during validation avoids trusting an older model value after a failed binding parse. Production forms should integrate field messages and a message summary consistently, rather than opening success dialogs for ordinary validation. The English literals keep this demonstration self-contained; replace them with the resource-model pattern before deployment.

Do not set the Input's native HTML number mode simply because the business value is numeric. Browser number-input behavior can conflict with localized grouping and decimal separators. Let the UI5 type handle display parsing. The type may normalize a valid value to a decimal string with fewer trailing zeros; if the transport contract requires exactly two digits, canonicalize that validated string explicitly before using the strict helper above. Do not canonicalize by converting it through `Number`.

Formatting with two displayed digits is not a financial rounding policy. Tax, exchange rates, allocations, and currency-specific rounding can require more scale and explicit rules. UI formatting is allowed to show a rounded display while retaining a more precise model value, but the application must not mistake that display for an authoritative posting amount.

## Exact fixed-scale addition without floating-point rounding

This independent module at `webapp/util/addAmounts.js` accepts the canonical nonnegative two-scale strings defined above. It requires `orderValues.js` and a modern browser with BigInt. It performs no rounding at all; it converts exact minor units, adds them, and returns another string. It must not add values of different currencies.

```javascript
sap.ui.define([
  "com/acme/orders/util/orderValues"
], function (orderValues) {
  "use strict";
  return function addAmounts(values) {
    if (!Array.isArray(values)) {
      throw new Error("INVALID_AMOUNTS");
    }
    let total = 0n;
    values.forEach(function (value) {
      if (!orderValues.validAmount(value)) {
        throw new Error("INVALID_AMOUNT");
      }
      const parts = value.split(".");
      total += BigInt(parts[0]) * 100n + BigInt(parts[1]);
    });
    const result = String(total / 100n) + "." +
      String(total % 100n).padStart(2, "0");
    if (!orderValues.validAmount(result)) {
      throw new Error("AMOUNT_OVERFLOW");
    }
    return result;
  };
});
```

For example, `addAmounts(["0.10", "0.20"])` returns `"0.30"` exactly. No binary approximation or `Math.round(value * 100)` is involved. BigInt values never enter JSONModel or JSON serialization; only the final string does. Empty input produces `"0.00"`, and a total exceeding the declared precision is rejected.

This helper is not a general accounting engine. It has no negative amounts, fractional minor units, currency conversion, or rounding modes. For those needs, use a backend domain operation or an approved exact-decimal implementation with explicit policy. Do not expand a short helper until it becomes an unreviewed financial subsystem.

## Calendar dates without timezone drift

Place this module at `webapp/util/orderFormatters.js`. It accepts Gregorian wire dates in `yyyy-MM-dd` form. Strict parsing plus a round trip rejects invalid dates such as February 30. UTC is used as an internal anchoring convention so a date-only value does not shift when displayed in a different timezone. Display style still follows the user's locale.

```javascript
sap.ui.define([
  "sap/ui/core/format/DateFormat",
  "sap/ui/core/CalendarType"
], function (DateFormat, CalendarType) {
  "use strict";

  function parseCalendarDate(value) {
    if (typeof value !== "string" ||
        !/^\d{4}-\d{2}-\d{2}$/.test(value)) {
      return null;
    }
    const wire = DateFormat.getDateInstance({
      pattern: "yyyy-MM-dd",
      calendarType: CalendarType.Gregorian,
      strictParsing: true,
      UTC: true
    });
    const date = wire.parse(value, true);
    return date && wire.format(date, true) === value ? date : null;
  }

  return {
    isCalendarDate: function (value) {
      return !!parseCalendarDate(value);
    },
    calendarDate: function (value) {
      const date = parseCalendarDate(value);
      return date ? DateFormat.getDateInstance({
        style: "medium",
        UTC: true
      }).format(date, true) : "";
    },
    statusState: function (status) {
      return status === "APPROVED" ? "Success" :
        status === "REJECTED" ? "Error" : "None";
    }
  };
});
```

To use it, import `com/acme/orders/util/orderFormatters` into a controller, expose it as `formatter: orderFormatters` in the controller definition, and bind `text="{path: 'orders>OrderDate', formatter: '.formatter.calendarDate'}"`. The `orders` context must contain the contract's ISO string. The local Orders controller can add that import and property without changing its CRUD methods.

This formatter is not suitable for an instant such as a shipment timestamp. Instants need a timezone policy, often the user's configured zone or the business location's zone. V2 date-only `Edm.DateTime` values can already be JavaScript Date objects and require a different type configuration; V4 `Edm.Date` is represented as a string. [models.md](./models.md) explains this distinction.

For editable V4 date strings, prefer `sap.ui.model.odata.type.Date` with an appropriate nullability constraint. Its formatting, parsing, and validation support the model representation. A display-only formatter cannot validate a user's localized date input. Always test leap days and timezone extremes rather than assuming an ISO-looking string is valid.

Creating DateFormat instances when invoked here favors correctness after localization changes over premature global caching. If profiling justifies caching, invalidate caches when relevant language, calendar, or timezone configuration changes. A singleton formatter created before a language switch can otherwise retain stale locale behavior.

## Debounce with explicit cancellation

This complete module at `webapp/util/debounce.js` reduces repeated read requests while typing. It does not cancel a request that has already started. Pair it with the generation and abort policy in [controllers.md](./controllers.md#latest-request-wins-remote-controller).

```javascript
sap.ui.define([], function () {
  "use strict";
  return function debounce(callback, delay) {
    let timer = null;
    function cancel() {
      if (timer !== null) {
        clearTimeout(timer);
        timer = null;
      }
    }
    function debounced() {
      const receiver = this;
      const args = Array.prototype.slice.call(arguments);
      cancel();
      timer = setTimeout(function () {
        timer = null;
        callback.apply(receiver, args);
      }, delay);
    }
    debounced.cancel = cancel;
    return debounced;
  };
});
```

Create the debounced function once during controller initialization, invoke it with the input value, and call `.cancel()` during exit. Do not pass the UI5 event object into a delayed callback; extract the needed string immediately. The callback must handle any promise rejection it creates because the debounce helper does not invent an error policy for asynchronous work.

Avoid a global debounced singleton shared by several controllers. Typing in one screen could cancel another screen's work, and captured controller references could keep destroyed views alive. Reuse the factory, not an instance whose lifecycle belongs to a particular screen.

## Manual verification and reuse checklist

- Validate `"000042"` without converting or changing its key representation.
- Reject blank names, malformed currency codes, noncanonical amounts, and precision overflow.
- Enter localized amounts under English and German settings; inspect canonical model strings.
- Enter invalid text after a valid amount; validation must not save the stale valid model value.
- Check exact addition, empty input, and overflow using known expected strings.
- Verify leap-day acceptance, invalid-day rejection, and date display near timezone boundaries.
- Change locale and confirm date and decimal display update appropriately.
- Confirm unknown status codes never receive an unjustified success state.
- Destroy a controller while debounce is pending; no callback should start afterward.
- Inspect payload projection and messages for unexpected fields or raw backend text.

These are manual acceptance scenarios for integration, not a claim that an application test suite was run.

## Official API references

Research access: the SDK host could not be resolved during research. API behavior was checked through SAP's official OpenUI5 GitHub sources instead. The [Decimal SDK reference](https://ui5.sap.com/#/api/sap.ui.model.odata.type.Decimal) and [DateFormat SDK reference](https://ui5.sap.com/#/api/sap.ui.core.format.DateFormat) are navigation links, not claims of successful SDK access; select your deployed version when consulting them.

- [OData Decimal string representation, parsing, and constraints](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/odata/type/Decimal.js).
- [OData Date representation and calendar handling](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/odata/type/Date.js).
- [DateFormat locale, strict parsing, and UTC options](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/core/format/DateFormat.js).
- [JSONModel updates and supported data behavior](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/json/JSONModel.js).
- [ResourceModel locale configuration](https://github.com/SAP/openui5/blob/1.120.0/src/sap.ui.core/src/sap/ui/model/resource/ResourceModel.js).
