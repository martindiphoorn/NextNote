# 3 Frontend angular 
During the setup part we have run: 

```bash
ng new nxt-note --style=scss
```

This created a new project called nxt-note and we are now going to extend it. 
We have add --style=scss to tell angular that we don't want to use old style sheets.

If you want to know more about scss style sheets check out the sass lang page.

https://sass-lang.com/guide

Watch out that you check the SCSS examples and not the SASS ones.
SCSS is compatible with normal css, but allows much more like nesting.


## 3.1 Components all over the place
Angular 6 uses components for everything. Navigation is also done by changing components.
Even the start of the application is a component. It is called app.component.ts 

So let's create a new component for our note list. Which gives us an overview of notes.
We use the angular cli to create a new component. Let's create our first component with:

```bash
ng generate component NoteList
```

this will create a functional folder called note-list with four files:

>* note-list.component.scss
>* note-list.component.html
>* note-list.component.spec.ts
>* note-list.component.ts

Scss is the sass stylesheet, html is the layout ts is the typescript file for the component.
And spec.ts files are used for unit testing and will not be distrubated.   

Feels good? Let's create our second component:

```bash
ng generate component Note
```

## 3.2 Routing for navigation

We also need to make it possible to reach the components. To do this we need to make some routing.
The basics can be generated with angular cli, let's run the following command:

```bash
ng generate module app-routing --flat --module=app
```
> --flat puts the file in src/app instead of its own folder.
> --module=app tells the CLI to register it in the imports array of the AppModule.

This will generate an app-routing.module.ts which is loaded from the app.module.ts.
So now we have a new module named AppRouting and it is loading by the app.module.ts.

Now let's open the app-routing.module.ts and make it a routing module, by adding some configuration and add some routes to our current note and note list components.

```typescript
import {NgModule} from '@angular/core';
import {RouterModule, Routes} from "@angular/router";
import {NoteListComponent} from "./note-list/note-list.component";
import {NoteComponent} from "./note/note.component";

const routes: Routes = [
  { path: '', redirectTo: '/notes', pathMatch: 'full' },
  { path: 'notes', component: NoteListComponent },
  { path: 'note/:id', component: NoteComponent }
];


@NgModule({
  imports: [ RouterModule.forRoot(routes) ],
  exports: [ RouterModule ]
})
export class AppRoutingModule { }
```

So let's explain this a bit. In the constant routes we define the paths to our components.
You see that the path to note contains an placeholder :id which can be used to open the note by it's id in the detail page.
The last line is the default redirect if nothing is given. The option pathMatch tells the router that is should be an exact match with nothing else behind it.

Next we create an import of the RouterModule by using forRoot with our routes constant.
forRoot is often used by modules that contain parts that should only live once during the application (singleton).
In our case we only want one RouterModule.

We still need to tell the RouterModule where to output the components we just navigated to.

Open app.component.html and change the contents to:

```html
<h1>{{title}}</h1>
<router-outlet></router-outlet>
```

The contents of ```{{title}}``` can be found in app.component.ts, open it and change the title to 'Next Notes'.

Now we can now test our routing by starting up docker compose and changing the url by hand.

> use ```./build.sh && docker-compose up``` in the docker folder to start it

Now open up a browser and type in: http://localhost:8080/   
If you have watch the url you should have noticed that it has changed to /notes.
Let's change it to /note/1 and we should see a page with: note works!

We just finished creating the base routing for our application.

Now we will add a link from the list to the note.
Let's open note-list.component.html and replace the contents with:

```html
<h2>Notes overview</h2>
<p>
  this is a test link to <a routerLink="/note/1">Note 1</a><br/>
  this is a test link to <a routerLink="/note/2">Note 2</a>
</p>
```

When we are in the note page we want to display the note id and also a link back to the notes.
Open note.component.html and change the content into:

```html
<h2>Note details</h2>
<a routerLink="/notes">Back to the overview</a>
<p>
  this will become the details page of our note. You reached this note with id: {{id}}
</p>
```

Next we need to open the note.component.ts and make the id available for the html.
The comments will try to explain the code, you can skip that.

```typescript
import { Component, OnInit } from '@angular/core';
import {ActivatedRoute} from "@angular/router";

@Component({
  selector: 'app-note',
  templateUrl: './note.component.html',
  styleUrls: ['./note.component.scss']
})
export class NoteComponent implements OnInit {
  // the html will retrieve this variable for displaying
  id: number;

  // variables in the constructor will be injected by angular
  // By making it private we can use it in the class with this.route
  // Without private it is only available in the constructor.
  constructor(private route: ActivatedRoute) { }

  // ngOnInit is a lifecycle hook and will be called when all injections/inputs have been handled.
  ngOnInit() {
    this.getNote();
  }

  // This is now just a way to set the id. But we will change this later to retrieve the note.
  getNote(): void {
    this.id = +this.route.snapshot.paramMap.get('id');
  }
}
``` 

Now rebuild and start your containers. 
> If you don't know how you can read back

## 3.3 Services for data retrieval
Before we can continue with creating or updating notes we first need to create the communication with the backend.
We will do this with injectable services. But first we need to add the HttpClient to our project.

Open app.module.ts and add the following lines:

```typescript
...
import {HttpClientModule} from "@angular/common/http";
...

@NgModule({
  ...

  imports: [
    BrowserModule,
    HttpClientModule,
    AppRoutingModule,
  ],
  
  ...
```

> The dots (...) means that we skipped lines, just add the missing lines to your file.

So now we can make http connections for data retrieval. Let's create our first service.
Again we use the cli to create the base. We put in the note/shared folder so we know it will be shared through te application.

```bash
ng generate service note/shared/NoteApi
```

Open the new note-api.service.ts file and change it to the following code.

```typescript
import {Injectable} from '@angular/core';
import {HttpClient} from "@angular/common/http";

@Injectable({
  providedIn: 'root'
})
export class NoteApiService {

  private API_URL = 'http://localhost:8080/notes';

  constructor(private  httpClient: HttpClient) {
  }

  public all() {
    return this.httpClient.get(this.API_URL);
  }

  public get(id: number) {
    return this.httpClient.get(this.API_URL + '/' + id);
  }
}
``` 

We now have a simple api which can be used to retrieve all or one note from the backend.
However there is no type safety. We don't know what will come back. Let's add some type safety.

First we need to create a note object in typescript. Create a note.model.ts in the note/shared folder with the following contents:

```typescript
export class Note {
  id: number;
  name: string;
}
```

Next we update the NoteApiService with the typesafety in place:

```typescript
import {Injectable} from '@angular/core';
import {HttpClient} from "@angular/common/http";
import {Note} from "./note.model";
import {Observable} from "rxjs";

@Injectable({
  providedIn: 'root'
})
export class NoteApiService {

  private API_URL = 'http://localhost:8080/notes';

  constructor(private  httpClient: HttpClient) {
  }

  public all(): Observable<Note[]> {
    return this.httpClient.get<Note[]>(this.API_URL);
  }

  public get(id: number): Observable<Note> {
    return this.httpClient.get<Note>(this.API_URL + '/' + id);
  }
}
``` 

Now we need to change the note-list and note components to retrieve the data and display it.

note-list.component.ts

```typescript
import { Component, OnInit } from '@angular/core';
import {NoteApiService} from "../note/shared/note-api.service";
import {Note} from "../note/shared/note.model";

@Component({
  selector: 'app-note-list',
  templateUrl: './note-list.component.html',
  styleUrls: ['./note-list.component.scss']
})
export class NoteListComponent implements OnInit {
  notes: Note[];

  constructor(private noteApiService: NoteApiService) { }

  ngOnInit() {
    this.noteApiService.all().subscribe(value => {
      this.notes = value;
    })
  }
}
```

note-list.component.html
```html
<h2>Notes overview</h2>
<ul>
  <li *ngFor="let note of notes" routerLink="/note/{{note.id}}">{{note.name}}</li>
</ul>

```

In angular there are two ways to handle forms. The first is the FormsModule which is similar to AngularJS with ngModel.
The other is ReactiveFormsModule which is more control based.
As most people know the FormsModule style from AngularJS we will use the ReactiveFormsModule here.

So we first need to import the ReactiveFormsModule in the app.module.ts:

app.module.ts
```typescript
...
import {ReactiveFormsModule} from "@angular/forms";
...

@NgModule({
  ...
  imports: [
    BrowserModule,
    HttpClientModule,
    ReactiveFormsModule,
    AppRoutingModule
  ]
  ...

``` 

note.component.ts
```typescript
import {Component, OnInit} from '@angular/core';
import {ActivatedRoute} from "@angular/router";
import {NoteApiService} from "./shared/note-api.service";
import {FormBuilder, FormGroup} from "@angular/forms";
import {Note} from "./shared/note.model";

@Component({
  selector: 'app-note',
  templateUrl: './note.component.html',
  styleUrls: ['./note.component.scss']
})
export class NoteComponent implements OnInit {
  editForm: FormGroup;

  constructor(
    private route: ActivatedRoute,
    private noteApiService: NoteApiService,
    private formBuilder: FormBuilder) {
  }

  ngOnInit() {
    this.getNote();
  }

  getNote(): void {
    const id = +this.route.snapshot.paramMap.get('id');

    // Request the service for the record and create the form
    this.noteApiService.get(id).subscribe(value => this.createForm(value));
  }

  createForm(note: Note) {
    // use the FormBuilder to create a FormGroup
    this.editForm = this.formBuilder.group(note);

    // Watch for changes in the form
    this.editForm.valueChanges.subscribe(value => {
      console.log(value);
    })
  }
  
}
```

note.component.html
```html
<h2>Note details</h2>
<a routerLink="/notes">Back to the overview</a>

<form [formGroup]="editForm" *ngIf="editForm">
  <label>
    id:
    <input type="number" formControlName="id" readonly>
  </label>
  <label>
    Name:
    <input type="text" formControlName="name">
  </label>
</form>

```

Can you do this for groups? 

## 3.4 Create/Update a note
Great we can display data, but for displaying we need data. 
Creating and updating is almost the same so we will do that in the same way.

First we will extend our service with two methods, create and update.
Next we will create a new button on the list for a new note.
And finally we add a save button to save the data.

Let's create the create and update methods. Add these to your existing service.

note-api.service.ts
```typescript
 
  public create(note: Note): Observable<Note> {
    return this.httpClient.post<Note>(this.API_URL, note);
  }

  public update(note: Note): Observable<Note> {
    return this.httpClient.put<Note>(this.API_URL + '/' + note.id, note);
  }
```

Add a new button to the note list. 

note-list.component.html
```html
<h2>Notes overview</h2>
<a routerLink="/note/0">new note</a>
<ul>
  <li *ngFor="let note of notes" routerLink="/note/{{note.id}}">{{note.name}}</li>
</ul>
```

Add a save button to the note. Just add this line to the end off the file.

note.component.html
```html
<button (click)="save()">save</button>
```

Next we need to implement the save, but also need to check if the id=0 to detect a new note.

note.component.ts
```typescript
   ...
   
   getNote(): void {
       const id = +this.route.snapshot.paramMap.get('id');
       if (id === 0) {
         // Create new note
         const note = new Note();
         note.id = 0;
         note.name = '';
         this.createForm(note);
       } else {
         // Retrieve the data
         this.noteApiService.get(id).subscribe(value => this.createForm(value));
       }
   }
   
   ...
   
   save() {
       const note = <Note> this.editForm.getRawValue();
       if (note.id === 0) {
         this.noteApiService.create(note).subscribe(value => {
           console.log('Created new node', value);
         });
       } else {
         this.noteApiService.update(note).subscribe(value => {
           console.log('Updated existing node', value);
         });
       }
   }

```

Did you notice the console.log after a succesfull create or update?
We probably want to go back to the list instead.

Let's change that replace the console.log statements or put this line below the console.log

note.component.ts
```typescript
this.router.navigate(['/notes']);
```

As we did not use router before we also add it to the constructor

note.component.ts
```typescript

...

import {ActivatedRoute, Router} from "@angular/router";

...

constructor(
    private route: ActivatedRoute,
    private router: Router,
    private noteApiService: NoteApiService,
    private formBuilder: FormBuilder) {
  }

```