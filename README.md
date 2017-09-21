# Introduction to LightDataTable
## nuget
https://www.nuget.org/packages/Generic.LightDataTable/
## What is LightDataTable
LightDataTable is an object-relation mappar that enable .NET developers to work with relations data using objects.
LightDataTable is an alternative to entityframwork. is more flexible and much faster than entity framework.
Even though it lacking database migration, its has a greater and more flexiable structure to work with.
## Can i use it to an existing database
Yes you could easily implement your existing modules and use attributes to map all your Primary Keys and Foreign Key without even
touching the database.
## Expression
LightDataTable has its own provider called SqlQueriable, which could handle almost every expression like Startwith,
EndWith Containe and so on
Se Code Example for more info.
## Code Example
let's start by creating the dbContext, lets call it Repository
```
    // Here we inherit from Transaction Data which contains the database logic for handling the transaction.
    // well thats all we need right now.
    public class Repository : TransactionData
    {
        /// <param name="appSettingsOrSqlConnectionString">
        /// AppSettingsName that containe the connectionstringName,
        /// OR ConnectionStringName,
        /// OR Full ConnectionString
        /// Default is Dbconnection
        /// </param>
        public Repository(string appSettingsOrSqlConnectionString = "Dbconnection") :
        base(appSettingsOrSqlConnectionString)
        {
        }

    }
```
let's start building our models, lets build a simple models User
```
    // Table attribute indicate that the object Name differ from the database table Name
    [Table("Users")]
    [Rule(typeof(UserRule))]
    public class User : DbEntity
    {
        public string UserName { get; set; }

        public string Password { get; set; }
        
        // Here we indicate that this attribute its a ForeignKey to object Role.
        [ForeignKey(type: typeof(Role))]
        public long Role_Id { get; set; }
        
        // when deleting an object the light database will try and delete all object that are connected to 
        // by adding IndependentData we let the lightdatatable to know that this object should not be automaticlly deleted
        [IndependentData]
        public Role Role { get; set; }

        public List<Address> Address { get; set; }
    }
    
    [Table("Roles")]
    public class Role : DbEntity
    {
        public string Name { get; set; }

        public List<User> Users { get; set; }
    }
    
    public class Address : DbEntity
    {
        public string AddressName { get; set; }
        // in the User class we have a list of adresses, lightDataTable will do an inner join and load the address 
        // if its included in the quarry
        [ForeignKey(typeof(User))]
        public long User_Id { get; set; }
    }
    
    // LightDataTable has its own way to validate the data.
    // lets create an object and call it UserRule
    // above the User class we have specified this class to be executed before save and after.
    // by adding [Rule(typeof(UserRule))] to the user class
    public class UserRule : IDbRuleTrigger
    {
        public void BeforeSave(ICustomRepository repository, IDbEntity itemDbEntity)
        {
            var user = itemDbEntity as User;
            if (string.IsNullOrEmpty(user.Password) || string.IsNullOrEmpty(user.UserName))
            {
                // this will do a transaction rollback and delete all changes that have happened to the database
                throw new Exception("Password or UserName can not be empty");

            }
        }

        public void AfterSave(ICustomRepository repository, IDbEntity itemDbEntity, long objectId)
        {
            var user = itemDbEntity as User;
            user.ClearPropertChanges();// clear all changes.
            // lets do some changes here, when the item have updated..
            user.Password = MethodHelper.EncodeStringToBase64(user.Password);
            // and now we want to save this change to the database 
            user.State = ItemState.Changed;
            // the lightdatatable will now know that it need to update the database agen.
        }
    }

```
## Quarry and Expression
Lets build some expression here and se how it works
```
   //To the header
   using Generic.LightDataTable;
   
   using (var rep = new Repository())
   {
        // LoadChildren indicate to load all children herarkie.
        // It has no problem handling circular references.
        // The quarry dose not call to the database before we invoke Execute or ExecuteAsync
        var users = rep.Get<User>().Where(x => 
                (x.Role.Name.EndsWith("SuperAdmin") &&
                 x.UserName.Contains("alen")) ||
                 x.Address.Any(a=> a.AddressName.StartsWith("st"))
                ).LoadChildren().Execute(); 
                
        // lets say that we need only to load some children and ignore some other, then our select will be like this instead
          var users = rep.Get<User>().Where(x => 
                (x.Role.Name.EndsWith("SuperAdmin") &&
                 x.UserName.Contains("alen")) ||
                 x.Address.Any(a=> a.AddressName.StartsWith("st"))
                ).LoadChildren(x=> x.Role.Users.Select(a=> a.Address), x=> x.Address)
                .IgnoreChildren(x=> x.Role.Users.Select(a=> a.Role)).OrderBy(x=> x.UserName).Skip(20).Take(100).Execute();        
        Console.WriteLine(users.ToJson());
        Console.ReadLine();
   }

```
## Edit, delete and insert
LightDataTable have only one method for insert and update.
It depends on primarykey, Id>0 to update and Id<=0 to insert.
```
   using (var rep = new Repository())
   {
        var users = rep.Get<User>().Where(x => 
                (x.Role.Name.EndsWith("SuperAdmin") &&
                 x.UserName.Contains("alen")) ||
                 x.Address.Any(a=> a.AddressName.StartsWith("st"))
                ).LoadChildren().Execute(); 
         foreach (User user in users.Execute())
         {
             user.UserName = "test 1";
             user.Role.Name = "Administrator";
             user.Address.First().AddressName = "Changed";
             // now we could do rep.Save(user); but will choose to save the whole list late
         }
        users.Save();    
        // how to delete 
        // remove all the data offcource Roles will be ignored here 
        users.Remove();
        // Remove with expression
        users.RemoveAll(x=> x.UserName.Contains("test"));
        // we could also clone the whole object and insert it as new to the database like this
        var clonedUser = users.Execute().Clone().ClearAllIdsHierarki();
        foreach (var user in clonedUser)
        {
            // Now this will clone the object to the database, of course all Foreign Key will be automatically assigned.
            // Role wont be cloned here. cource it has IndependedData attr we could choose to clone it too, by invoking
            // .ClearAllIdsHierarki(true);
            rep.Save(user);
        }
        Console.WriteLine(users.ToJson());
        Console.ReadLine();
   }

```
## Issues
This project is under developing and it's not in its final state so please report any bugs or improvement you find
