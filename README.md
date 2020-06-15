# Witness

![GitHub Workflow Status](https://img.shields.io/github/workflow/status/MrZoidberg/Witness/Build) [![Codacy Badge](https://app.codacy.com/project/badge/Grade/69696e45f75f455e9a7654ca2b12f227)](https://www.codacy.com/manual/mihail.merkulov/Witness?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=MrZoidberg/Witness&amp;utm_campaign=Badge_Grade) ![Nuget](https://img.shields.io/nuget/v/WitnessValidation)

Witness is a small vaidation library for .NET Core that allows building object validation in a fluent style.
It is mostly targets custom validation rules that are very hard to implement in existing solutions like [Fluent Validation](https://github.com/FluentValidation/FluentValidation).

Features:

- Fully fluent-style based on pure functions
- DI inside validation rules
- Conditional branches with `.If`

## Get Started

Witness can be installed using the Nuget package manager or the dotnet CLI.

`dotnet add package WitnessValidation`

## Example

```csharp
using Witness;

public class Person
{
    public string FirstName { get; set; }

    public string LastName { get; set; }
    
    public uint Age { get; set; }
    
    public string GithubAccount { get; set; }
}


public static class PersonValidationRulesExtensions
{    
    [Pure]
    public static Func<IValidationContext<Person, string>> ShouldExistInGithub(this Func<IValidationContext<Person, string>> context)
    {
        return () =>
        {
            var c = context();

            bool isValid = !string.IsNullOrEmpty(c.OUV);
            if (isValid)
            {
                using HttpClient client = new HttpClient();
                var result = new TaskFactory(TaskScheduler.Current)
                    .StartNew(async () => await client.GetAsync($"https://github.com/{c.OUV}"))
                    .Unwrap()
                    .GetAwaiter()
                    .GetResult();

                isValid = result.StatusCode != HttpStatusCode.NotFound;
            }

            if (!isValid)
            {
                c.ValidationErrors.Add($"{c.OUVName} should be a valid Github username");
            }

            return c;
        };
    }
}

Person person = new Person()
{
    FirstName = "Joe",
    LastName = string.Empty,
    Age = 5,
    GithubAccount = "test",
};

RootValidationContext<Person> validationContext = new RootValidationContext<Person>(person);
var result = person
    .SetupValidation()
    .RuleFor(c => c.FirstName).ShouldNotBeEmptyOrNull()
    .And()
    .RuleFor(c => c.LastName).ShouldNotBeEmptyOrNull()
    .And()
    .RuleFor(c => c.Age).ShouldBeInRange(18, 100)
    .And()
    .RuleFor(c => c.GithubAccount).ShouldExistInGithub()
    .ExecuteValidation();
```
