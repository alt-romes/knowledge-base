Removing all errors from the desugarer (HsToCore errors)

Multiplicity coercions should work similarly to representation coercions do see `DsMultiplicityCoercionsNotSupported`.

We want to move the logic to the typechecker and definitely not have it in the desugarer

---

We were getting regressions in linear types compile-time so Richard improved other parts of the compiler to even it out.

---

Use the `design-document` tag in the mega issue tracker
We really need to convince Simon that this is all worth doing, and that it'll make the code clearer and less buggy