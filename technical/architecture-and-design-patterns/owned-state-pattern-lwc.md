# Owned State Pattern (lwc)

{% hint style="info" %}
When to use this pattern: You have shared, mutable state that flows through a deeply nested component tree, and mutations from any component at any depth must be immediately visible to every other component in the tree.
{% endhint %}

### Purpose

The owned state pattern applies unidirectional data flow, the same principle behind React's lifting state up, Redux's single store, and the Gang of Four's mediator pattern, to Lightning Web Components.

A single owner component holds the canonical state, passes it down via `@api`, and receives mutations back through `CustomEvent`. Every change produces a new object reference, which is the only mechanism LWC provides to trigger downstream re-renders of complex objects.

### The Problem: Shared Mutable State in a Deep Tree

Some UIs are simple: a form, a few inputs, a submit button. The data lives in one component and never leaves. LWC handles this effortlessly.

Other UIs are not simple. Consider any experience where:

* A deeply nested component tree, five, six, seven levels deep, shares a single data model.
* Multiple screens or views all operate on the same underlying structure.
* A mutation in one branch must be immediately reflected in every other branch.
* The data model itself is recursive or hierarchically nested.
* The same structure must serialize to and from the server without transformation.

This shape shows up in a variety of contexts:

* **Shopping carts**: a product browser, a mini-cart, and a checkout flow all share the same model. Adding an item recalculates discounts, tax, and totals across every panel.
* **Multi-step wizards**: a multi-screen form where step four’s summary depends on data entered in step one. The user jumps back, changes something, and every subsequent step reflects it immediately.
* **Product configurators**: a base product with options that enable or disable other options, while a preview panel and price breakdown stay in sync with every change.
* **Quote builders**: nested line items with configurable quantities, pricing tiers, and optional add-ons. Adjusting one line recalculates the section total, the overall quote, and the projected margin.
* **Multi-view dashboards**: the same dataset rendered as a table, a chart, and a detail panel. An action in one view updates the others instantly.

Without a deliberate strategy for shared mutable state, you end up in one of two places: a tangle of cross-component messaging where nobody can trace which event caused which update, or a pile of duplicated state where components slowly drift out of sync.

The implementation example below uses a shopping cart, but the pattern is identical for all of the above.

### The Solution: Established Patterns, Applied to LWC

The front-end ecosystem solved this problem years ago. The answer is **unidirectional data flow**: state flows down, events flow up, and mutations produce new state rather than modifying existing state in place.

Every major framework has its own implementation of this idea, and we draw directly from them.

**React: Lifting State Up.** When two React components need to share state, the official guidance is to move that state to their nearest common ancestor. The ancestor passes data down as props. Children communicate changes upward via callbacks. The ancestor re-renders, and the new props flow back down to everyone.

This is the foundation of our approach: one owner at the top, data down via `@api`, changes up via events.

**Redux / Flux: Single Store with Immutable Reducers.** Redux takes lifting state up to its logical conclusion: all shared state lives in a single immutable store. Components dispatch actions describing what happened. Reducer functions take the current state plus the action and return a brand-new state object. The store replaces its reference, and every connected component re-renders.

Our model class is the store. Our mutation methods are reducers. Our events are dispatch.

**Mediator Pattern.** In the Gang of Four’s mediator pattern, a central object coordinates communication between components that do not know about each other. Children never reference siblings. They communicate only through the mediator.

The owner component plays exactly this role: a product grid and a checkout summary never talk to each other directly. They dispatch events upward to the owner, and the owner pushes new state downward to both.

**Event Sourcing.** Our clone-and-return approach produces a new snapshot on every mutation. A sequence counter on the model is a lightweight version of an event sequence number, useful for debugging race conditions and optimistic concurrency.

The vocabulary changes across ecosystems, but the mechanic is always the same: one owner, data down, events up, new state on every change.

#### How We Apply This to LWC

LWC does not ship with a state management framework. The building blocks are there: `@api` for data down, `CustomEvent` for events up, and reference identity for change detection.

But there is no higher-level abstraction that wires them together the way React Context or Redux does. We use these primitives to implement the same unidirectional flow that other frameworks provide out of the box.

The owned state pattern has three parts:

1. **A plain JavaScript class** models the shared state. Not an LWC component, not a module-scoped variable, not a store. A class with fields and methods, the equivalent of a Redux store and its reducers in a single object.
2. **A single owner** at the top of the component tree holds the one canonical instance and passes it down via `@api`. This is lifting state up: one component is the source of truth, everything else receives.
3. **An event contract** flows changes upward. Children call the model’s own mutation methods, which return new instances, then dispatch an event carrying the new instance. The owner adopts it, and LWC’s reactivity pushes it back down to everyone.

At any point in time, one instance exists and every component in the tree points to the same object. When a mutation happens, the new instance replaces the old one as the single shared reference.

Because the model is a full JavaScript class with methods, it can handle calculations that would otherwise require a server round-trip. Adding an item, recalculating a subtotal, applying a discount rule, validating a configuration constraint, these all run client-side inside the mutation methods.

The server is only needed when the client genuinely cannot do the work: persisting data, running pricing engines with server-side rules, or applying security-sensitive logic.

This is a direct benefit of the mediator pattern. Without a central coordinator, each component that needs updated state tends to make its own server call.

With the owner acting as mediator and the model handling calculations in its mutation methods, one client-side operation replaces what would otherwise be multiple independent callouts.

The trade-off is **prop drilling**, a term from the React ecosystem for passing data through every intermediate component between the owner and the leaf that needs it. Frameworks like Redux and React Context exist partly to avoid it. LWC has no equivalent shortcut, so every component in the chain must explicitly receive the model and relay events.

The benefit is that the data flow is fully explicit. You can open any component, read its template, and trace exactly where the model comes from and where events go.

#### Summary

* We apply **unidirectional data flow** to LWC using its native primitives: `@api`, `CustomEvent`, and reference identity.
* Model the shared state as a **plain JavaScript class** with mutation methods that return new instances, like Redux reducers.
* The **top-level component** owns the single instance and passes it down via `@api`, lifting state up.
* The owner acts as a **mediator**: children never reference siblings, they communicate through the owner.
* Children **call the model’s own methods**, then **bubble an event** to notify the owner.
* Client-side mutation methods **reduce backend callouts**: the server is only called when the client genuinely cannot do the work.
* Every mutation returns a **new instance** because that is the only way to trigger child `@api` setters and re-renders.

### When to Use This Pattern

This pattern solves a specific category of problem. It adds ceremony to compensate for capabilities LWC does not provide natively, so it should not be a default choice.

Reach for it when the complexity of the problem justifies the overhead.

#### Good Fit

* **Deeply nested component trees**: when the distance between the mutation source and the consumers is three or more levels, passing individual properties and coordinating events becomes unmanageable.
* **Multiple screens operating on the same model**: a multi-step wizard, a checkout flow, a configuration builder where each screen reads and writes the same underlying data structure. The model survives screen transitions because the owner sits above all of them.
* **Recursive or hierarchically nested data**: models where items contain sub-items, which may contain their own sub-items. Trees, not tables.
* **Mutations in one place affect calculations elsewhere**: adding an item recalculates discounts, which recalculates tax, which updates the total. The clone-and-return approach ensures all cascading effects are captured in a single new object.
* **Serialization boundaries**: when the same data structure goes to and from the server without transformation.
* **Audit and debugging needs**: a sequence counter on the model lets you track how many mutations have occurred, which is valuable for debugging race conditions and optimistic concurrency.

#### Poor Fit

* **Simple forms and single-screen components**: if the data lives in one component and does not need to be shared, standard `@track` properties are simpler and sufficient.
* **Flat data**: a model with five primitive fields and no nesting does not justify the ceremony of a class with `from()` and mutation methods. Individual `@api` properties work fine.
* **Cases where LMS is the right tool**: when sibling components live in different parts of a Lightning page with no shared ancestor, Lightning Message Service is designed for exactly that. The owned state pattern requires a shared component tree.
* **Shallow trees where `@api` properties suffice**: if the distance between producer and consumer is one or two levels, passing individual properties is clearer and cheaper.
* **Read-only data**: if components only need to display shared data and never mutate it, a simpler data-down approach without the event contract is enough.

{% hint style="warning" %}
The owned state pattern adds ceremony: event relay handlers at every intermediate level, a model class with clone-and-return methods, and a team convention with no framework enforcement. If your data is flat and your tree is shallow, that ceremony is overhead without payoff.
{% endhint %}

### Implementation: A Shopping Cart Example

To make the pattern concrete, we walk through a shopping cart, a multi-screen experience where a product browser, a cart sidebar, a checkout flow, and a pricing engine all share the same cart state.

Configuration builders, multi-step wizards, and order management flows follow the same structure.

#### Part 1: The Model as a Plain JavaScript Class

The shared state is a plain JavaScript class with fields, a `static from()` factory that creates deep copies, and a `serialize()` method for the server boundary.

```jsx
export class Cart {
	items = [];
	discountCodes = [];
	subtotal = 0;
	taxTotal = 0;
	discountTotal = 0;
	updateSequenceId = 0;

	constructor() {
		this.items = [];
		this.updateSequenceId = 0;
	}

	static from(json) {
		if (typeof json === 'string') {
			json = JSON.parse(json);
		}

		let result = new Cart();
		result.items = json.items
			? json.items.map(item => CartItem.from(item))
			: [];
		result.discountCodes = json.discountCodes
			? [...json.discountCodes]
			: [];
		result.subtotal = json.subtotal || 0;
		result.taxTotal = json.taxTotal || 0;
		result.discountTotal = json.discountTotal || 0;
		result.updateSequenceId = json.updateSequenceId || 0;
		return result;
	}

	serialize() {
		return JSON.stringify(this);
	}
}
```

The critical design decision is `static from()`. This factory method creates a deep copy from JSON or from an existing instance.

Every child object (line items, discount details, nested sub-items) has its own `from()` method, so the entire tree is reconstructed. No shared references survive.

#### Part 2: Clone-and-Return Mutations

Every method that modifies the model creates a new instance first:

```jsx
removeProduct(product) {
	let result = Cart.from(this);
	result.items = this.items.filter(item => item.id !== product.id);
	return result;
}

upsertProduct(product) {
	let result = Cart.from(this);
	result.items = [];
	let isUpdate = false;

	this.items.forEach(item => {
		if (item.id === product.id) {
			result.items.push(CartItem.updateProduct(item, product));
			isUpdate = true;
		} else {
			result.items.push(item);
		}
	});

	if (!isUpdate) {
		result.items.push(CartItem.updateProduct(new CartItem(), product));
	}

	return result;
}

setDiscountCodes(codes) {
	let result = Cart.from(this);
	result.discountCodes = codes;
	return result;
}
```

The shape is consistent: `from(this)` at the top, mutations on the copy, `return result` at the bottom.

#### Why Not Mutate in Place?

Passing a complex object via `@api` means we have already stepped outside LWC’s built-in reactivity for that property. The framework does not deeply observe objects. It only tracks reference identity.

We replace the missing reactivity with two mechanisms, one for each direction:

* **Upward (child to parent):** a `CustomEvent` replaces reactivity. The child tells the parent “the model changed” by dispatching an event with the new instance.
* **Downward (parent to children):** a **new object reference** is the only way to trigger child `@api` setters. LWC has no `forceUpdate()`. When the owner assigns `this._cart = event.detail.cart`, every child’s `@api` setter fires only if the reference is different from what it already holds.

If a child mutated the model in place and fired a payload-less event, the owner would catch it but would have no way to push the change downward.

Assigning `this._cart = this._cart` is a no-op to LWC because the reference has not changed. Child `@api` setters would not fire. Components that recalculate in their setter would never update. The DOM would show stale data.

The new instance solves the downward leg. The event solves the upward leg. Both are needed because LWC gives us neither direction for free when working with complex objects.

#### Part 3: Children Mutate, Then Notify

Children call methods on the model directly. The model encapsulates its own mutation logic.

Because every method returns a new instance, the child gets a fresh reference to assign locally and pass up via the event.

```jsx
export default class LineItemEditor extends LightningElement {
	@api
	get cart() {
		return this._cart;
	}
	set cart(value) {
		this._cart = value;
	}

	_cart;

	handleDelete(event) {
		// Call the model's own method -- returns a NEW instance
		this._cart = this._cart.removeProduct({ id: event.detail });

		// Notify the parent: "here is the updated model"
		this.dispatchEvent(
			new CustomEvent('changecart', {
				detail: { cart: this._cart }
			})
		);
	}
}
```

The owner, the top-level component, catches these events and reassigns, which triggers LWC’s reactivity to push the new reference down to every child:

```jsx
export default class Storefront extends LightningElement {
	_cart = new Cart();

	handleCartChange(event) {
		this._cart = event.detail.cart;
	}
}
```

```html
<c-product-browser
	cart={_cart}
	onchangecart={handleCartChange}
></c-product-browser>

<c-checkout-view
	cart={_cart}
	onchangecart={handleCartChange}
></c-checkout-view>
```

The key insight:

* **The model knows how to change itself** (via its methods).
* **The component tree knows how to propagate that change** (via events and the owner’s reassignment).

These are separate responsibilities.

### Component Hierarchy and Event Flow

Here is a simplified tree showing how the pattern looks in practice. The owner sits at the top. Every branch receives the same instance.

Events flow upward. New references flow downward.

```
storefront (owns _cart)
	|
	+-- browseView (receives cart, dispatches changecart)
	|	|
	|	+-- productGrid (receives cart, dispatches changecart)
	|	|	|
	|	|	+-- productCard (dispatches addtocart)
	|	|
	|	+-- miniCart (receives cart, dispatches changecart)
	|		|
	|		+-- lineItem (dispatches remove / quantitychange)
	|
	+-- cartView (receives cart, dispatches changecart)
	|	|
	|	+-- lineItem (dispatches remove / quantitychange)
	|	+-- discountEntry (dispatches changecart)
	|	+-- orderSummary (reads cart)
	|
	+-- checkoutView (receives cart, dispatches changecart)
		|
		+-- shippingForm (reads cart)
		+-- paymentForm (reads cart)
		+-- orderReview (reads cart, dispatches changecart)
```

Notice the **multiple screens**: the browse view, cart view, and checkout view are different views within the same experience.

A user adds an item on the browse screen, reviews it on the cart screen, and completes the order on the checkout screen, but all three share the same cart instance because the owner sits above them.

The model survives screen transitions without serializing state between steps.

#### Tracing a Mutation Through the Tree

A user clicks the remove button on a line item inside the mini-cart while browsing.

1. The **line item** dispatches a `remove` event with the item ID.
2. The **mini-cart** catches it, calls `this._cart.removeProduct({ id: event.detail })`. The model’s own method returns a new instance.
3. The **mini-cart** assigns the new instance to its local `_cart` and dispatches `changecart` with it.
4. The **browse view** receives the event, assigns the new instance, and re-dispatches upward.
5. The **storefront** receives the event and sets `this._cart = event.detail.cart`.
6. LWC’s reactive system detects the reference change on `_cart` and pushes the new value down to every child simultaneously. The product grid, the mini-cart, and the checkout screen all update in the same render cycle.

The mutation originated deep in one branch, but every component across every screen reflects the same consistent state.

The event chain exists solely because LWC’s reactivity is top-down: the owner must reassign for the change to propagate.

### Comparing Alternatives

#### vs. Lightning Message Service (LMS)

LMS is designed for cross-DOM communication: sibling components that live in different parts of a Lightning page with no shared ancestor.

It works well for that use case.

For a deeply nested component tree that shares a single data model, LMS introduces problems:

* **Implicit dependencies**: any component can subscribe to any channel, making it hard to trace who reacts to what.
* **No consistency guarantee**: subscribers process messages independently, so during a render cycle, some components may show stale data while others show the update.
* **Debugging difficulty**: when something goes wrong, you cannot follow the data flow by reading the template hierarchy.

The owned state pattern keeps the data flow explicit. Open any component, look at the template, and you can see exactly where data comes from and where events go.

#### vs. Passing Individual Properties

A model with 15 or more properties could be decomposed into individual `@api` properties on every child.

This creates three problems:

* **Massive API surfaces**: every component that touches the model needs a dozen `@api` properties, and adding a new field means updating every component in the chain.
* **Coordination overhead**: when multiple properties change at once (adding an item changes both `items` and `subtotal`), you need to ensure both arrive before the child re-renders.
* **Loss of cohesion**: the model is a single concept. Splitting it across properties fragments the mental model.

Passing a single object keeps each component’s API clean: one property in, one event out.

#### vs. Module-Scoped State

A tempting shortcut is to export a mutable object from a shared module and import it everywhere.

```jsx
// DO NOT do this
export const sharedState = { items: [], subtotal: 0 };
```

This fails for two reasons:

* **Bypasses reactivity**: LWC does not observe changes to module-scoped variables. Mutating `sharedState.items.push(item)` will not trigger any re-renders.
* **No ownership**: any component can write to the shared object at any time, making it impossible to reason about when or why the state changed.

The owned state pattern works _with_ the framework’s reactivity system. Every change flows through an assignment (`this._model = newModel`), which is exactly what LWC is built to detect.

{% hint style="warning" %}
Module-scoped mutable state is the most common source of “ghost bugs” in LWC applications: components that show stale data with no obvious cause. If you find yourself importing a shared mutable object, step back and consider whether the owned state pattern is a better fit.
{% endhint %}

### Recursive Data Models

One reason this pattern pairs well with complex experiences is recursive data. Many real-world models are not flat lists. They are trees where items contain other items.

A bundled product is not a single line item. It is a parent that contains child products, each with their own price, quantity, and configuration. This creates a tree, not a table:

```
Cart
	+-- Bundle A (parent product)
	|	+-- Included Item 1 (bundled, included in parent price)
	|	+-- Included Item 2 (bundled, included in parent price)
	|	+-- Included Item 3 (bundled, included in parent price)
	|
	+-- Product B (parent product)
	|	+-- Add-On 1 (priced separately)
	|	+-- Add-On 2 (priced separately)
	|
	+-- Product C (standalone, no children)
```

The line item class mirrors this recursion. It has its own `from()` factory, its own mutation methods, and its own sub-items array.

When the top-level `Cart.from()` runs, it recursively constructs every node in the tree. No shared references survive at any depth.

This matters because operations at one level affect calculations at others. Removing a bundled item might invalidate the bundle discount on the parent. Adding an add-on changes the parent line’s total, which changes the cart’s subtotal, which changes the discount calculation, which changes the tax total.

The clone-and-return approach ensures all cascading effects are captured in a single new object.

The same recursive shape appears in configuration builders where options contain sub-options, quote builders where line items contain nested add-ons, and multi-step forms where sections contain repeatable sub-sections.

### The Server Mirror

The JavaScript class is a deliberate mirror of a server-side DTO. The same properties, the same nested structure, the same naming.

```jsx
// Receiving from the server
const cart = Cart.from(serverResponse);

// Sending to the server
const serialized = cart.serialize(); // JSON.stringify(this)
```

The `static from()` method doubles as a deserializer. When the server returns a JSON representation of the model, `from()` reconstitutes it into a fully functional client-side object with all its methods intact.

This eliminates the transformation layer that typically sits between server responses and client-side state. There is no mapping function, no adapter class, no shape conversion. The DTO _is_ the model.

When the structure changes, a new field, a new nested type, you update the server DTO and the JavaScript class in parallel.

The `from()` method handles defaults for missing fields, so the change is backward-compatible by default.

### Not Every Component Needs the Model

Passing the full model down to every component in the tree is not the goal. The pattern applies to **orchestrating components**, the ones that coordinate state across the experience.

Leaf components that render a single, focused piece of UI should receive only the data they need via individual `@api` properties.

Consider a product card that displays a name, price, and quantity stepper. It does not need to know about discount codes, tax calculations, or what other items are in the cart:

```html
<c-product-card
	name={item.label}
	price={item.price}
	quantity={item.quantity}
	onquantitychange={handleQuantityChange}
></c-product-card>
```

```jsx
export default class ProductCard extends LightningElement {
	@api name;
	@api price;
	@api quantity;

	handleIncrement() {
		this.dispatchEvent(
			new CustomEvent('quantitychange', { detail: this.quantity + 1 })
		);
	}

	handleDecrement() {
		if (this.quantity > 0) {
			this.dispatchEvent(
				new CustomEvent('quantitychange', { detail: this.quantity - 1 })
			);
		}
	}
}
```

The parent, which does have the model, handles the translation between the shared state and the component’s simple API:

```jsx
handleQuantityChange(event) {
	const updatedProduct = { ...this.product, quantity: event.detail };
	this._cart = this._cart.upsertProduct(updatedProduct);
	this.dispatchEvent(
		new CustomEvent('changecart', { detail: { cart: this._cart } })
	);
}
```

This separation has real benefits:

* **Reusability**: a product card that takes `name`, `price`, and `quantity` can be used anywhere, not just inside the shared-state flow.
* **Testability**: testing a component with three primitive `@api` properties is trivial compared to constructing a full model instance in a test harness.
* **Performance**: LWC can skip re-rendering a child component when its individual `@api` values have not changed, even if the parent’s model reference has.
* **Clarity**: the component’s API documents exactly what it needs.

#### The Rule of Thumb

Ask: **Does this component need to read from or write to multiple parts of the shared state?**

If yes, pass the model.

If it only renders one item or handles one interaction, pass the specific values it needs.

Components that typically receive the full model:

* Layout coordinators (the storefront, the checkout view, the wizard container)
* Summary panels that aggregate across items
* Calculation engines that read items and write totals

Components that should receive individual `@api` properties:

* Cards, line item rows, quantity steppers
* Price formatters, discount badges
* Sliders, input controls
* Any component you want to reuse outside the shared-state context

The model flows through the **spine** of the component tree. The **leaves** get only what they need.

### Trade-Offs

| **Benefit**                                       | **Cost**                                                 |
| ------------------------------------------------- | -------------------------------------------------------- |
| Consistent state across all components            | Deep event chains through the component hierarchy        |
| Predictable reactivity via reference changes      | Memory allocation for each mutation (clone-and-return)   |
| Explicit, traceable data flow                     | Boilerplate event handlers at each intermediate level    |
| Clean `@api` surface (one prop in, one event out) | Every intermediate component must relay the event        |
| Seamless server DTO mirroring                     | Requires discipline: the team must follow the convention |
| Survives screen transitions in multi-step flows   | Pattern is overkill for simple, flat, single-screen data |

There is also a broader consideration: LWC does not enforce or assist with this pattern. There is no linter rule that catches a missed event relay, no type system that validates the model shape, and no framework hook that enforces the clone-and-return convention.

The pattern holds together through team convention and code review. In frameworks where state management is a first-class feature, much of this enforcement comes for free.

The relay boilerplate is the most common complaint. Every component between the leaf and the owner needs a handler that assigns the new instance and re-dispatches the event.

This is mechanical and repetitive. We consider it a worthwhile trade for the explicitness it provides. There is never a question about how a change reaches the owner.

### Key Takeaways

1. **Model your shared state as a plain class**, not as a component, a store, or a module variable.
2. **Make every mutation return a new instance**: `from(this)`, mutate the copy, return it.
3. **Designate a single owner** at the top of the tree and pass the instance down via `@api`.
4. **Use events to flow changes upward**: children dispatch, parents relay, the owner assigns.
5. **Mirror the server DTO** so serialization is trivial in both directions.
6. **Pass the model through the spine, not the leaves**: leaf components get individual `@api` properties for reusability and performance.
7. **Know what you are adopting**: this is an advanced pattern built on top of primitives that LWC was not specifically designed to support in this combination.

It solves real problems for deep trees, recursive data, and multi-screen flows. For flat data and shallow trees, simpler approaches win.
