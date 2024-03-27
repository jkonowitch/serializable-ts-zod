# Serializable TS

This library provides a robust framework for serializing and deserializing JavaScript objects in a declarative way, leveraging ES6 decorators and `zod` for schema definitions and validations. It ensures strict type safety and runtime checks, making it especially useful when working with other type-safe libraries.

## Features

- **Schema Definition with `zod`**: Allows for defining strict schemas for data validation and type inference.
- **Type Safety**: Ensures type safety during serialization and deserialization processes, supporting full type inference on serialization return types.
- **Serialization and Deserialization**: Facilitates the conversion between class instances and plain objects, supporting deep serialization of nested objects and collections.
- **Validation**: Provides mechanisms for validating class properties both at instantiation and during property updates.

## ES6 Decorators

This library takes advantage of new features of the JavaScript language, specifically ES6 decorators, which are currently at the [stage 3 proposal status](https://github.com/tc39/proposal-decorators) and are fully implemented in Typescript 5. Decorators offer a syntactically convenient way to modify class properties and methods, making them ideal for defining serialization and validation rules in a declarative manner. By leveraging decorators, Serializable TS provides a clean and intuitive API for marking class properties as serializable and attaching validation logic directly within class definitions.

## Why would I use this?

This library is particularly useful in the following scenarios:

- **Domain-Driven Design (DDD) Applications**: When building domain-driven applications, you often want to separate your core domain logic from the persistence mechanism (e.g., databases, APIs). This library allows you to define your domain models as serializable classes, abstracting away the persistence details and enabling you to work with plain JavaScript objects for storage or transmission.

- **Any Application with Abstracted Persistence**: Even if you're not following a strict DDD approach, this library can be beneficial in any application where you want to decouple your core logic from the persistence layer. By defining your models as serializable classes, you can easily convert them to and from plain objects, facilitating integration with various storage or transmission mechanisms.

- **Type-Safe Data Validation**: The library leverages `zod` for schema definitions and validations, ensuring strict type safety and runtime checks. This is particularly useful when working with other type-safe libraries or when dealing with complex data structures that require robust validation.

## Getting Started

### Installation

You can install the library using npm (or your package manager of choice). You must also have `zod` installed as a peer dependency.

```bash
npm install serializable-ts zod
```

### Defining Schemas and Creating Classes

To ensure a seamless developer experience, **classes must define a public readonly SCHEMA property**, referencing the corresponding zod schema. Additionally, classes **must use this SCHEMA as the constructor argument for instantiation**.

```typescript
import { z } from 'zod';

const AddressSchema = z.object({
  details: z.object({ city: z.string(), zipCode: z.string() })
});

class Address {
  public readonly SCHEMA = AddressSchema;

  @serializable('details')
  protected accessor details: { city: string; zipCode: string };

  constructor(parameters: z.infer<typeof AddressSchema>) {
    this.details = parameters.details;
  }
}
```

### Serialization

The `serialize` function converts class instances into plain objects, using schemas for type inference and structure validation.

```typescript
const address = new Address({ details: { city: 'City', zipCode: '12345' } });
const serializedAddress = serialize(address);
```

### Deserialization

The `deserialize` function converts plain objects back into class instances, ensuring the data matches the defined schemas.

```typescript
const deserializedAddress = deserialize(serializedAddress, Address);
```

## Advanced Serialization

### Serializable Properties

When dealing with properties that are instances of classes marked as serializable, the library seamlessly handles their serialization. This is particularly useful for composing complex data models where some properties are objects with their own serialization logic.

Consider the following simplified `Person` class implementation:

```typescript
const PersonSchema = z.object({
  address: z.instanceof(Address),
  name: z.string()
});

class Person {
  public readonly SCHEMA = PersonSchema;

  @serializable('address', Address)
  accessor address: Address;

  @serializable('name')
  accessor name: string;

  constructor(params: z.infer<typeof PersonSchema>) {
    this.address = params.address;
    this.name = params.name;
  }
}
```

In this example, the `Person` class has an `address` property which is an instance of the `Address` class, itself a serializable entity. The library correctly handles the serialization of this nested object according to its defined schema and serialization rules.

### Collection Serialization

When serializing collections, such as arrays of objects, the library allows for the definition of custom serialization and deserialization logic for each item in the collection. This is particularly useful for managing lists of serializable entities.

Here's how you can define a `Company` class that contains a list of `Person` instances:

```typescript
const CompanySchema = z.object({
  people: z.array(z.instanceof(Person))
});

class Company {
  public readonly SCHEMA = CompanySchema;

  @serializable('people', {
    doSerialize: (people) => people.map(serialize),
    doDeserialize: (people) => people.map((person) => deserialize(person, Person))
  })
  accessor people: Person[];

  constructor(params: z.infer<typeof CompanySchema>) {
    this.people = params.people;
  }
}
```

In this `Company` class example, the `people` property is an array of `Person` instances. The library uses the provided custom serialization (`doSerialize`) and deserialization (`doDeserialize`) functions to process each `Person` object in the array, ensuring they are correctly serialized and deserialized according to their own schemas.

### Usage Example

Here's how you might use these classes together:

```typescript
const address = new Address({ details: { city: 'CityName', zipCode: 'PostalCode' } });
const person = new Person({ name: 'John Doe', address });
const company = new Company({ people: [person] });

// Serialize the company object, including nested Person and Address objects
const serializedCompany = serialize(company);

// The type of 'serializedCompany' is fully inferred from the serialization process.
// We infer the structure of the serialized object based on the schemas
// used in the class definitions. In this case, 'typeof serializedCompany'
// would be inferred as follows:
// typeof serializedCompany === {
//   people: {
//     name: string;
//     address: {
//       details: {
//         city: string;
//         zipCode: string;
//       };
//     };
//   }[];
// }

// Deserialize back into a Company object from the serialized data
const deserializedCompany = deserialize(serializedCompany, Company);
```

In this example, serializing the `company` object automatically handles the serialization of the nested `Person` and `Address` objects. Similarly, deserializing from `serializedCompany` reconstructs the full `Company` instance, along with all nested objects, preserving the structure and types as defined by their schemas.

### Subclassing and Overrides

Subclasses can extend base classes, overriding serialization behavior and adapting to more specific schemas.

```typescript
class InternationalAddress extends Address {
  public readonly SCHEMA = InternationalAddressSchema;
  @serializable('country')
  accessor country: string;
}
```

### Validation

Properties can be validated using `zod` schemas with the `@validateWith` and `@validateSetWith` decorators. This allows for [self-encapsulation](https://martinfowler.com/bliki/SelfEncapsulation.html) without a lot of boilerplate. Note there are two separate decorators, one for `accessors` and the other for `setters`. This can be combined with serialization (or not - it is technically a standalone feature).

```typescript
const EmailSchema = z.object({
  address: z.string().email(),
  foo: z.string().min(4)
});

class Email {
  public readonly SCHEMA = EmailSchema;

  @validateWith(EmailSchema.shape.address)
  @serializable('address')
  accessor address: string;

  #foo: string;

  @serializable('foo')
  get foo() {
    return this.#foo;
  }

  @validateSetWith(EmailSchema.shape.foo)
  set foo(s: string) {
    this.#foo = s;
  }
}
```

## Testing

The library's functionality is thoroughly tested using unit tests. Please review them [here]() to see all of this functionality in action.
