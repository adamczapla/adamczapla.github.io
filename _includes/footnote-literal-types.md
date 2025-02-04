[^1]: **What are Literal Types?**

    A literal type in C++ is a type that can be used in a `constexpr` context, meaning inside `constant expressions`. This includes:
    * Built-in types such as `int`, `char`, `double`, `bool`, and `nullptr_t`
    * Enumerations (`enum` and `enum class`)
    * `Pointer` types to literal types, including `const` and `nullptr_t` pointers
    * `Pointers to members` of literal types
    * `Literal classes`[^2]

[^2]: **Requirements for a class to be a `literal class`**

    * All `non-static` members must be literals.
    * The class must have at least one user-defined `constexpr` constructor, or all `non-static` members must be initialized `in-class`.
    * `In-class` initializations for `non-static` members of `built-in` types must be `constant expressions`.
    * `In-class` initializations for `non-static` members of class types must either use a user-defined `constexpr` constructor or have no     initializer. If no initializer is given, the default constructor of the class must be `constexpr`. Alternatively, all `non-static` members of the class must be initialized `in-class`.
    * A `constexpr` constructor must initialize every member or at least those that are not initialized `in-class`.
    * Virtual or normal default destructors are allowed, but user-defined destructors with {} are not allowed. User-defined     constructors with {} are allowed if they are declared as `constexpr`. However, user-defined `constexpr` destructors in literal classes are often of limited use because literal classes do not manage dynamic resources. In `non-literal` classes, however, they can be important, especially for properly deallocating dynamic resources in a `constexpr` context.
    * `Virtual` functions are allowed, but `pure virtual` functions are not.
    * `Private` and `protected` member functions are allowed.
    * `Private` and `protected` inheritance are allowed, but `virtual` inheritance is not.
    * `Aggregate classes`[^4] with only literal `non-static` members are also considered literal classes. This applies to all aggregate classes without a base class or if the base class is a literal class.
    * `Static` member variables and functions are allowed if they are `constexpr` and of a literal type.
    * `Friend` functions are allowed inside literal classes.
    * Default arguments for constructors or functions must be `constant expressions`.
    
    > A literal type ensures that objects of this type can be evaluated at compile time, as long as all dependent expressions are `constexpr`. 
    {: .prompt-info }

[^3]: **Requirements for a class to be a `aggregate class`**
    
    1. **What is allowed:**
    * `Public` members
    * `User-declared` destructor
    * `User-declared` copy and move assignment operators
    * Members can be `not literals`
    * `Public` inheritance
     *  `Protected` or `private` **static** members
    
    2. **What is not allowed:**
    * `Protected` and `private` **non-static** members
    * `User-declared` constructors
    * `Virtual` destructor
    * `Virtual` member functions
    * `Protected/private` or `virtual` inheritance
    * Inherited constructors (by `using` declaration)
    
    3. **Restrictions for the base class:**
    * Only `public` **non-static** members allowed OR
    * `Public` constructor required for `non-public` **non-static** members
