---
title: "Dependency Injection"
author: Eric Luper
date: '2022-06-17'
draft: false
---

# A Brief on Dependency Injection

Dependency Injection, also known as Inversion of Control, is a design pattern used to loosely couple dependent objects. It makes the fifth principle of the SOLID design principles.

SOLID:

- S : Single-responsiblity Principle
- O : Open-closed Principle
- L : Liskov Substitution Principle
- I : Interface Segregation Principle
- D : Dependency Inversion Principle

Considering web development and the practice of dependency injection when handling objects making http requests, an example of the code simplification of dependency injection can be given by considering entities with mail services.

Different entities may be defined with the ability to send or recieve mail to or from some outside source. We may have something large and complicated, like a business, with the ability to send and recieve mail, or define individual people or households. We could define the logic for handling and recieving mail within each entitiy.

- Business
    - Logic for sending requests
- Household
    - Logic for sending requests
- Person
    - Logic for sending requests

This is duplicative, and logistics is not trivial. The whole process of mail delivery will have to be described for each entity. If a change is made (maybe persons and households have upgraded from carrier pidgeons to vehicle delivery), it will have to be made for multiple entities. One solution may be inheritance, however the aim is loose coupling and reuse. Realistically, persons and households, and maybe even businesses, do not really care what is involved in the process of delivery. Just that things are delivered. The dependency injection solution would be to create a service. A mail delivery service.

- Mail Delivery Service
    - Logic for sending requests

- Business
    - Passes address and request to mail delivery service.
- Household
    - Passes address and request to mail delivery service.
- Person
    - Passes address and request to mail delivery service.

All an entity needs to care about is its address or maybe address it wants to send something to, and what it wants to send or recieve. All concerns (playing into seperation of concerns here) of the process of delivery belong to the mail service.

# Dependency Injection in C#

Dependency Injection is paired with abstraction. In C#, interfaces are used opposed to concrete classes in loosely coupling dependencies, making it easier for testing as mock objects implementing the interface can be used. For a backend (Enitity Framework Core) using the repository design pattern, we set up a repo service to pull the database context out of a user controller, and create an abstraction that would let us use a mock repo for testing if needed.

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

    public async Task<User> Get(int Id)
    {
      return await _context.users.Include(i => i.Contributions).ThenInclude(c => c.ContributionType).FirstOrDefaultAsync(i => i.Id == Id);
    }
  }
}
```

To set up a service, this line is added to the Program.cs.
```c#
builder.Services.AddTransient<IUserRepo, UserRepo>();
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
    private readonly FromDTO _fromDTO;
    private readonly IUserRepo _userRepo;

    public UserController(IUserRepo UserRepo)
    {
      _userRepo = UserRepo;
      _toDTO = new ToDTO();
      _fromDTO = new FromDTO();
    }

    [HttpGet]
    public async Task<ActionResult<IEnumerable<UserDTO>>> GetAll()
    {
      var userList = await _userRepo.GetAll();
      var userDTOList = userList.Select(u => _toDTO.ToUserDTO(u)).ToList();
      return userDTOList;
    }

    [HttpGet("{Id}")]
    public async Task<ActionResult<UserContDTO>> Get(int Id)
    {
      var user = await _userRepo.Get(Id);
      if (user == null)
      {
        return NotFound();
      }
      return _toDTO.ToUserContDTO(user);
    }
  }
}
```

# Dependency Injection in Angular

For the free company website [council-of-yggdrasil.company](https://council-of-yggdrasil.company/home), the homepage contains a section that displays FFXIV news and maintenaince information using the Lodestone API. The model of the desired json response is defined in a lodestone.interface.ts.

## lodestone.interface.ts
```ts
export interface LodestoneTopic {
    url: string;
    time: string;
    image: string;
    description: string;
}

export interface Companion {
    url: string;
    title: string;
    start: string;
    end: string;
    current: boolean;
}

export interface LodestoneMaintenance {
    companion: Companion[];
}
```

A service dependent on this model is created to provide methods for making http requests. This is done in lodestone.service.ts. Notice that this service is further dependent on HttpClient. Dependency injection allows the service to focus cleanly on just defining Get methods.

## lodestone.service.ts
```ts
import { HttpClient } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';
import { LodestoneMaintenance, LodestoneTopic } from '../model/lodestone.interface';
import { environment } from 'src/environments/environment';

@Injectable({
  providedIn: 'root'
})
export class LodestoneService {

  constructor(private http: HttpClient) { }

  /**
   * news/topics Get request that retrieves a list of current topics.
   * @returns An observable of topics.
   */
  getTopics(): Observable<LodestoneTopic[]> {
    return this.http.get<LodestoneTopic[]>(`${environment.lodestoneUrl}/topics`);
  }

  /**
   * news/maintenance/current Get request that retrieves a list of current topics.
   * @returns An observable of current maintenance.
   */
  getMaintenance(): Observable<LodestoneMaintenance> {
    return this.http.get<LodestoneMaintenance>(`${environment.lodestoneUrl}/maintenance/current`);
  }
}
```

The Home Component injects the lodestone dependency, allowing it to make requests for Lodestone information. Should other components need to make requests, this dependency could be injected to them as well. Should changes need to be made, such as the endpoint, this can be done in the single service itself. Should the json response be changed, just the model needs to be updated. This is the advantage of seperation of concerns.

## home.component.ts
```ts
import { Component, OnInit } from '@angular/core';
import { map } from 'rxjs';
import { LodestoneMaintenance, LodestoneTopic } from 'src/app/model/lodestone.interface';
import { LodestoneService } from 'src/app/services/lodestone.service';

@Component({
  selector: 'coy-home',
  templateUrl: './home.component.html',
  styleUrls: ['./home.component.scss']
})
export class HomeComponent implements OnInit {

  lodestoneTopics: LodestoneTopic[] = [];

  lodestoneMaintenance: LodestoneMaintenance = {
    companion: []
  };

  constructor(private lodestoneService: LodestoneService) { }

  ngOnInit(): void {
    this.lodestoneService.getTopics().pipe(
      map(e => {
        e = e.splice(0, 3);
        e.forEach(e => e.time = (new Date(e.time)).toString());
        return e;
      }))
      .subscribe(lodestoneTopics => this.lodestoneTopics = lodestoneTopics);

    this.lodestoneService.getMaintenance().pipe(
      map(e => {
        e.companion[0].start = new Date(e.companion[0].start).toString();
        e.companion[0].end = new Date(e.companion[0].end).toString();
        return e;
      })
    ).subscribe(lodestoneMaintenance => this.lodestoneMaintenance = lodestoneMaintenance);
  }
}
```

## Note on Abstraction in Typescript
One may have noticed that, in the Angular example, an interface was not defined for the injected LodestoneService. The service seemed have been injected as a concrete class opposed to how services are usually injected in C#. However, Typescript types are not concrete, and as long as a type follows the same structure as another type, that type will be accepted. Consider the example:

```ts
class LodestoneService {
  total: number;
}

class LodestoneServiceMock {
  total: number;
}

function showTotal(obj: LodestoneService) {
  console.log("total: " + obj.total);
}

const testObj: LodestoneServiceMock = { total: 5 };

showTotal(testObj); // Would throw a compiler error in C#, works in typescript
```

Because LodestoneServiceMock follows the same structure as LodestoneService, it is accepted as an input to function showTotal(obj: LodestoneService). This implies some existing abstraction Typescript has with its types that does not need to be defined explicitly as in C#.