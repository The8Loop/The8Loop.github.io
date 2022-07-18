---
title: "Cooking Passwords: Hashing and Salting"
author: E.S. Luper
date: 2022-07-18T08:53:14-05:00
draft: false
---

# Cooking Passwords: Hashing and Salting

## Why Cook Passwords
Sometimes, passwords get leaked. In 2020, Instagram suffered a databreach that leaked thousands of passwords in plain text. While it is often said that users have a responsibiliy to maintain password security (long, complicated passwords, regularly updated, unshared, not duplicated, etc.), services also have a responsibility towards their users to store their passwords in a secure manner. Plain text is not secure. Sometimes, passwords get leaked, and when the saved format is raw plain text, that's it. The user is completely compromised at the fault of the company. The basic solution is to not save passwords raw on the database, but to cook them. As a bare minimum, passwords should be hashed and salted.

### Hashing
To chop into small pieces, or more appropriate, to confuse or muddle, hashing passwords is the use of a one-way, deterministic encryption algorithm to convert a plain text password into a complex string. The resulting hash cannot be used to log in with, and cannot be trivially, if even, converted back to the user's original password. When users log in with their actual passwords, the log-in process will hash their entered password and compare it to the stored hash. In the event that a company has a password leak, Mr. Crook has to deal with a list of indecipherable strings that represent user passwords, rather than having the benefit of actual access to the plain text passwords.

```c#
Hash(password) = "70617373776f7264"
```

As effective as hashing is, simply hashing a password with no additional processing makes for a bland cooking. A leaked list of hashes can still be used to compromise accounts, such as with the use of a rainbow table. Attackers make a list of common passwords, and then run the list through hashing algorithms to create lists of resulting hashes. With these lists, attackers can find matches in the leaked password hashes to figure out a user's actual password. It is thus important for users to practice making complex passwords so as to avoid being on an attacker's dictionary. It is also important for companies to take additional steps in password cooking to minimize the risk of rainbow attacks by cooking with a little salt.

### Salting
Salting passwords is the introduction of a randomized string to a password upon creation, essentially enforcing some complexity to the password with the intent of keeping it off an attacker's dictionary. The salted password is then hashed. It is important to note that the salt is saved in the database with the hash. If the database is breached, the attacker does have the salt. It is a weakness of salts, but should not be viewed as a necessarily detrimental one. Security isn't always about making something completely fool-proof, which seldom is possible, but truly is about statistics and buying time for response. Salts, while saved, keeps an attacker's dictionary from an all-application to the hashed list. An attacker would have to update their rainbow table for every individual's salt, buying significant time for a security response. The nature of salt storage means it is still important for users to maintain complex passwords.

```c#
Hash(password + "864647499") = "239604f3e036306"
```

### Peppering
Peppers (or 'secret salt') are similar to salts, being a random string applied to a user password on creation. The difference between salt and pepper is that a pepper's value is hard-coded rather than saved on the database, and a single, static pepper is used for all created passwords, while salts are dynamically generated for every new password. The static and wide nature of a pepper means that maintainability is an issue should a pepper need to be updated, meaning they are generally avoided.

## Implementation for Council of Yggdrasil

[Council of Yggdrasil](https://council-of-yggdrasil.company/home) uses .NET's Rfc2898DeriveBytes to hash provided passwords. The provided code shows two end-points, the first for the signing up of a user, and the second for validating a log-in. Note that for passing log-in information, a post request should be used over a get request, since get requests display information in the url.

### Adding a User

```c#
[HttpPost]
    public async Task<ActionResult<User>> Post(UserSignupDTO userSignupDTO)
    {
      const int SALT_SIZE = 24; // size in bytes
      const int HASH_SIZE = 24; // size in bytes
      const int ITERATIONS = 100000; // number of pbkdf2 iterations

      // Generate a salt
      var provider = RandomNumberGenerator.Create();
      byte[] salt = new byte[SALT_SIZE];
      provider.GetBytes(salt);

      // Generate the hash
      Rfc2898DeriveBytes pbkdf2 = new Rfc2898DeriveBytes(userSignupDTO.Password, salt, ITERATIONS);

      userSignupDTO.Password = Convert.ToHexString(pbkdf2.GetBytes(HASH_SIZE));
      var user = _fromDTO.FromUserDTO(userSignupDTO, Convert.ToHexString(salt));
      try
      {
        await _userRepo.Post(user);
      }
      catch
      {
        return StatusCode(409); //Return 409 Conflict if username already exists.
      }
      return CreatedAtAction(nameof(Get), new { Id = user.Id }, user);
    }
```

### Checking Log-In Credentials

```c#
[HttpPost("Login")]
    public async Task<ActionResult<Boolean>> GetUserLogin(UserSignupDTO userLogin)
    {
      User user = await _userRepo.GetUserLogin(userLogin.Name);
      if (user == null)
      {
        return false;
      }

      const int HASH_SIZE = 24; // size in bytes
      const int ITERATIONS = 100000; // number of pbkdf2 iterations

      //Get salt in byte[]
      byte[] salt = Convert.FromHexString(user.Salt);

      // Generate the hash
      Rfc2898DeriveBytes pbkdf2 = new Rfc2898DeriveBytes(userLogin.Password, salt, ITERATIONS);
      userLogin.Password = Convert.ToHexString(pbkdf2.GetBytes(HASH_SIZE));

      return (user.Password == userLogin.Password);
    }
```