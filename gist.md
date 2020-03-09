NPoco/PetaPoco stored procedures with named strong type parameters
==
**StoredProcedures.tt** file automatically generates a C# code file with calsses and methods corresponding to your database stored procedure definitions. This is then used to simply call stored procedures within NPoco/PetaPoco and avoid magic strings and parameter type guessing. It also supports **output** parameters.

Stored procedures naming conventions
--
In order to have your stored procedure calls well structured there are particular yet simple naming conventions you should follow when creating your stored procedures:

1. Name your stored procedures as `ClassName_Method`
2. If a particular stored procedure shouldn't be *parsed* you should omit underscore character in its name; this will make it *private* from the perspective of your C# as it will only be accessible to other stored procedures but won't be parsed.

And that's it really. T4 will pick up all matching stored procedures, read their information and generate C# calling code with correctly typed and named parameters.

What about types and their compatibility?
--
There's a subset of all available types that this parser understands and uses. I've included most commonly used SQL types. If you'd like to add additional ones, simply add new ones to `typeMapper` dictionary.

All SQL types map to nullable value or reference C# equivalent types to satisfy possible value nullability. Parser can't know whether particular parameters can be nullable or not, therefore all generated C# code uses nullable typed parameters. This means that i.e. `bit` will be mapped to `bool?`, `datetime` to `DateTime?`, but `nvarchar` to `string` as it's a nullable (reference) type already.

End result
--
After you configure SQL database connection string (found early in T4 template in code line 37) parser generates static classes with matching static methods that return `NPoco.Sql` object instances that can be used with your C# directly when manipulating data. This means that it doesn't generate any new NPoco methods, but rather parameters for them. Majority of NPoco/PetaPoco methods have several overloads, one of them accepting `Sql` type parameter. This parser takes advantage of this fact.

Example
==
Suppose we write a stored procedure that creates a new user and then returns the newly generated record back so we can know user's ID right after it's been created (along with all other data). In modern web applications users usually get an activation email to activate their account and prove their email ownership.

```tsql
create procedure dbo.User_Create (
    @FirstName nvarchar(50),
    @LastName nvarchar(50),
    @Email nvarchar(200),
    @PasswordHash char(60),
    @ActivationCode char(32) out
)
as
begin
    -- generate new activation code
    set @ActivationCode = ...

    -- insert user with activation code
    ...

    -- return newly created user
    select Id, FirstName, LastName, Email
    from dbo.User
    where Id = @@scope_identity;
end
go
```

Parser will generate `Create` method as part of a `User` class that can have several other methods as well if there are other `User_...` stored procedures.

```csharp
internal static class StoredProcedures
{
    // other classes

    internal static partial class User
    {
        // other User methods
        
        public static Sql Create(string firstName, string lastName, string email, string passwordHash, out SqlParameter activationCode)
        {
            activationCode = new SqlParameter("@ActivationCode", SqlDbType.Char);
            activationCode.Direction = ParameterDirection.Output;
            activationCode.Size = 32;
            
            Sql result = Sql.Builder.Append(";exec dbo.[User_Create] @FirstName, @LastName, @Email, @PasswordHash, @ActivationCode out", new {
                FirstName = firstName,
                LastName = lastName,
                Email = email,
                PasswordHash = passwordHash,
                ActivationCode = activationCode
            });
            
            return result;
        }
        
        // other User methods
    }
    
    //other classes
}
```

You would then call/use this code in your C# code like so this:

```csharp
// created NPoco/PetaPoco IDatabase instance
using (var db = Database.GetDatabase())
{
    SqlParameter generatedCode;
    
    // password has already been hashed elsewhere above in code
    User result = db.Single<User>(StoredProcedures.User.Create(firstName, lastName, email, passwordHash, out generatedCode));
    
    // use generatedCode to likely send user an activation email
    this.SendEmail(email, generatedCode.Value);
    
    return result;
}
```

I've deliberately provided a more complex example using an output parameter so you can see how it's used. Majority of stored procedures usually use input parameters only and calling code is much simpler as all you have to do is provide appropriate parameters and use them.
