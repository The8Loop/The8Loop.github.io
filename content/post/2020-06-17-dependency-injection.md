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