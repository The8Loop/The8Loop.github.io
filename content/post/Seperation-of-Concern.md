---
title: "Seperation of Concern"
author: E.S. Luper
date: 2022-07-08T08:58:02-05:00
draft: false
---

# Seperation of Concerns

Following up on dependency injection and the use of the repository structure, a core principle of these approaches is seperation of concern. In building an application, it is a good practice to seperate the concerns, tasks, functionalities, between different classes and components. Angular supports the practice of seperation of concern very well with its component system. Not utilized, one may end up with very large page components that try to handle everything. For maintainibility, and reuse of repeated features, it becomes clear that things should be broken down into smaller components. In web development, and very well in other fields of software development, testing is of large interest. Seperating tasks into specific components increases testability.

When first learning to build a controller in .Net, one handles database access via the controller. With the repository structure, the concern of database access is seperated from the controller. Likewise, the repository should only be dealing with direct domain objects, and DTO mapping will be handled by a seperate class.

## IUserRepo and It's Implementation
```c#
using Microsoft.EntityFrameworkCore;
using CoYBackend.Models;

namespace CoYBackend.Services
{
  public interface IUserRepo
  {
    Task<User> Get(int Id);
    Task<IEnumerable<User>> GetAll();
  }

  public class UserRepo : IUserRepo
  {
    private readonly CoYBackendContext _context;

    public UserRepo(CoYBackendContext context)
    {
      _context = context;
    }

    public async Task<IEnumerable<User>> GetAll()
    {
      return await _context.users.ToListAsync();
    }
  }
}
```

## User Controller with Endpoints
```c#
using Microsoft.AspNetCore.Mvc;
using CoYBackend.Models;
using CoYBackend.Services;

namespace CoYBackend.Controllers
{
  [Route("api/[controller]")]
  [ApiController]
  public class UserController : ControllerBase
  {
    private readonly ToDTO _toDTO;
    private readonly IUserRepo _userRepo;

    public UserController(IUserRepo UserRepo)
    {
      _userRepo = UserRepo;
      _toDTO = new ToDTO();
    }

    [HttpGet]
    public async Task<ActionResult<IEnumerable<UserDTO>>> GetAll()
    {
      var userList = await _userRepo.GetAll();
      var userDTOList = userList.Select(u => _toDTO.ToUserDTO(u)).ToList();
      return userDTOList;
    }
  }
}
```

## To DTO Mapping
``` c#
using CoYBackend.Models;

namespace CoYBackend.Services
{
  public class ToDTO
  {
    public UserDTO ToUserDTO(User u)
    {
      var userDTO = new UserDTO()
      {
        Id = u.Id,
        Name = u.Name,
        IsActive = u.IsActive
      };
      return userDTO;
    }
  }
}
```
\
By seperating the DTO mapping into a class, it is made reusable and keeps mapping logic in one place. By pulling out the database context from the controller into a repository object, testing of the controller is made possible since a mock repository supplying mock domain objects can be injected into the controller in place of the real repository.