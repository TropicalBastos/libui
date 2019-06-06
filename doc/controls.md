<!-- 29 may 2019 -->

# Controls

## Overview

TODO

## Reference

### `uiControl`

```c
typedef struct uiControl uiControl;
uint32_t uiControlType(void);
#define uiControl(obj) ((uiControl *) uiCheckControlType((obj), uiControlType()))
```

`uiControl` is an opaque type that describes a control.

`uiControlType()` is the type identifier of a `uiControl` as passed to `uiCheckControlType()`. You rarely need to call this directly; the `uiControl()` conversion macro does this for you.

### `uiControlVtable`

```c
typedef struct uiControlVtable uiControlVtable;
struct uiControlVtable {
	bool (*Init)(uiControl *c, void *implData, void *initData);
	void (*Free)(uiControl *c, void *implData);
};
```

`uiControlVtable` describes the set of functions that control implementations need to implement. When registering your control type, you pass this in as a parameter to `uiRegisterControlType()`. Each method here is required.

Each method takes at least two parameters. The first, `c`, is the `uiControl` itself. The second, `implData`, is the implementation data pointer; it is the same as the pointer returned by `uiControlImplData(c)`, and is provided here as a convenience.

Each method is named for the `uiControl` function that it implements. As such, details on how to implement these methods are documented alongside those functions. For instance, instructions on implementing `Free()` are given under the documentation for `uiControlFree()`. The only exception is `Init()`, which is discussed under `uiNewControl()` below.

### `uiRegisterControlType()`

```c
uint32_t uiRegisterControlType(uiControlVtable *vtable, uiControlOSVtable *osVtable, size_t implDataSize);
```

`uiRegisterControlType()` registers a new `uiControl` type with the given vtables and returns its ID as passed to `uiNewControl()`. `implDataSize` is the size of the implementation data struct that is created by `uiNewControl()`.

`uiControlVtable` describes the functions of a `uiControl` common between platforms, and is discussed on this page. `uiControlOSVtable` describes functionst hat vary from OS to OS, and are described in the respective OS-specific uiControl implementation pages.

It is a programmer error to specify `NULL` for either vtable. An `implDataSize` of 0 is legal; the implementation data pointer will be `NULL`. This is not particularly useful, however. It is also a programmer error to specify `NULL` for any of the methods in either vtable — that is, all methods are required.

### `uiCheckControlType()`

```c
void *uiCheckControlType(void *c, uint32_t type);
```

`uiCheckControlType()` checks whether `c` is a `uiControl`, and if so, whether it is of the type specified by `type`. If `c` is `NULL`, or if either of the above conditions is false, a programmer error is raised. If the conditions are met, the function returns `c` unchanged.

This function is intended to be used to implement a macro that converts an arbitrary `uiControl` pointer into a specific type. For instance, `uiButton` exposes its type ID as a function `uiButtonType()`, and provides the macro `uiButton()` that does the actual conversion as so:

```c
#define uiButton(c) ((uiButton *) uiCheckControlType((c), uiButtonType()))
```

### `uiNewControl()`

```c
uiControl *uiNewControl(uint32_t type, void *initData);
```

`uiNewControl()` creates a new `uiControl` of the given type with the given data.

This function is meant for control implementations to use in the implementation of dedicated creation functions; for instance, `uiNewButton()` calls `uiNewControl()`, passing in the appropriate values for `initData`. Normal users should not call this function.

It is a programmer error to pass an invalid value for either `type` or `initData`.

**For control implementations**: This function allocates both the `uiControl` and the memory for the implementation data, and then passes both of these allocations as well as the value of `initData` into your `Init()` method. Return `false` from the `Init()` method if `initData` is invalid; if it is valid, initialize the control and return `true`. To discourage direct use of `uiNewControl()`, you should generally not allow `initData` to be `NULL`, even if there are no parameters. Do **not** return `false` for any other reason, including other forms of initialization failures; see [Error handling](error-handling.md) for details on what to do instead.

### `uiControlFree()`

```c
void uiControlFree(uiControl *c);
```

`uiControlFree()` frees the given control.

If `c` has children, those children are also freed. It is a programmer error to free a control that is itself a child of another control.

If `c` has any registered events, those event handlers will be set to be no longer run via `uiEventInvalidateSender()`. The registered handlers themselves will not be removed, to avoid the scenario of another `uiControl` being created with the same pointer value later triggering your handler unexpectedly.

It is a programmer error to specify `NULL` for `c`.

**For control implementations**: This function calls your vtable's `Free()` method. Parameter validity checks are already performed, `uiControlEventOnFree()` handlers have been called, and `uiControl`-specific events have been invalidated. Your `Free()` should invalidate any events that are specific to your controls, call `uiControlFree()` on all the children of this control, and free dynamically allocated memory that is part of your implementation data. Once your `Free()` method returns, libui will take care of freeing the implementation data memory block itself.