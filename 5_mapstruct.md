# 5 MapStruct

## What is MapStruct?
MapStruct is a code generator based on annotations and an interface.
Often there is a need to copy properties from one class to the other.
If there are many Data Transfer Objects (DTO) in your project this is an big helper. 
These interfaces are called mappers.

## Setup maven

Add an property to your pom.xml with the version of mapstruct. Just to be sure also check if you have java.version property.

```xml
<properties>
    ...
    <java.version>11</java.version>
    <org.mapstruct.version>1.3.0.Final</org.mapstruct.version>
    ...
</properties>
```

Next add the maven dependency for MapStruct.

```xml
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>${org.mapstruct.version}</version>
</dependency>
```

We also need to configure an annotationProcessor. This will make MapStruct run during compile time.
When you configure one annotationProcessor you also need to add lombok.
Root cause is that maven does not scan the classpath any more if paths are defined.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.8.1</version>
    <configuration>
        <source>${java.version}</source>
        <target>${java.version}</target>
        <annotationProcessors>
            <annotationProcessor>lombok.launch.AnnotationProcessorHider$AnnotationProcessor</annotationProcessor>
            <annotationProcessor>org.mapstruct.ap.MappingProcessor</annotationProcessor>
        </annotationProcessors>
        <annotationProcessorPaths>
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>${lombok.version}</version>
            </path>
            <path>
                <groupId>org.mapstruct</groupId>
                <artifactId>mapstruct-processor</artifactId>
                <version>${org.mapstruct.version}</version>
            </path>
        </annotationProcessorPaths>
        <compilerArgs>
            <arg>-Amapstruct.defaultComponentModel=spring</arg>
        </compilerArgs>
    </configuration>
</plugin>
```

The compiler argument is needed for MapStruct it will add @Component to the generated classes so we can AutoWire it.

## Creating a NoteMapper

Before creating the NoteMapper we first need an DTO which can be filled. This is our current Note Entity.

Note.java

```java
package com.example.nextnote.note;

import javax.persistence.*;

import com.example.nextnote.group.Group;

import lombok.Data;

@Data
@Entity
@Table(name = "note")
public class Note {
	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private Long id;

	@ManyToOne(fetch = FetchType.LAZY)
	@JoinColumn(name = "groupId")
	private Group group;

	private String name;
}
```

Let's create the NoteDto beside Note.java (in the same package).

NoteDto.java
```java
package com.example.nextnote.note;

import lombok.Data;

@Data
public class NoteDto {
	private Long id;
	private String name;
}
```

And now the mapper which will copy the properties from the entity to the dto and back.

NoteMapper.java
```java
package com.example.nextnote.note;

import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.MappingTarget;
import org.mapstruct.Mappings;

@Mapper
public interface NoteMapper {
	@Mappings({
			@Mapping(target = "id", ignore = true)
	})
	void toEntity(NoteDto noteDto, @MappingTarget Note note);

	NoteDto toDto(Note note);

    List<NoteDto> toDto(List<Note> note);
}
```

That's it. This will create an mapper class which copies all the properties and will ignore the id when it goes back to an entity
You can even view the generated source in the target/generated-sources folder.

NoteMapperImpl.java
```java
package com.example.nextnote.note;

import java.util.ArrayList;
import java.util.List;
import javax.annotation.processing.Generated;
import org.springframework.stereotype.Component;

@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2019-12-09T15:53:13+0100",
    comments = "version: 1.3.0.Final, compiler: javac, environment: Java 11.0.5-ea (Ubuntu)"
)
@Component
public class NoteMapperImpl implements NoteMapper {

    @Override
    public void toEntity(NoteDto noteDto, Note note) {
        if ( noteDto == null ) {
            return;
        }

        note.setName( noteDto.getName() );
    }

    @Override
    public NoteDto toDto(Note note) {
        if ( note == null ) {
            return null;
        }

        NoteDto noteDto = new NoteDto();

        noteDto.setId( note.getId() );
        noteDto.setName( note.getName() );

        return noteDto;
    }

    @Override
    public List<NoteDto> toDto(List<Note> note) {
        if ( note == null ) {
            return null;
        }

        List<NoteDto> list = new ArrayList<NoteDto>( note.size() );
        for ( Note note1 : note ) {
            list.add( toDto( note1 ) );
        }

        return list;
    }
}
```

As you can see above the class is annotated with @Component. That allow us to use it by injection.
Let's change our NoteController to return dto objects.

We only want to communicate with the dto objects. So we need to update all the methods and make use of our new mapper.
In the code below you will find the mapper in use. Check the difference with your old code.

```java
package com.example.nextnote.note;

import java.util.List;
import java.util.Optional;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
public class NoteController {

	private final NoteMapper noteMapper;

	private final NoteRepository noteRepository;

	/**
	 * Spring will automatically inject the mapper and the repository
	 *
	 * @param noteMapper
	 * @param noteRepository
	 */
	@Autowired
	public NoteController(NoteMapper noteMapper, NoteRepository noteRepository) {
		this.noteMapper = noteMapper;
		this.noteRepository = noteRepository;
	}

	/**
	 * Returns all our notes from the database
	 *
	 * @return
	 */
	@RequestMapping(value = "/notes", method = RequestMethod.GET)
	public List<NoteDto> all() {
		List<Note> notes = this.noteRepository.findAll();
		return noteMapper.toDto(notes);
	}

	/**
	 * Return the note with the specified id or null if is not available
	 *
	 * @param id
	 * @return
	 */
	@RequestMapping(value = "/notes/{id}", method = RequestMethod.GET)
	public NoteDto one(@PathVariable("id") Long id) {
		return this.noteRepository.findById(id).map(noteMapper::toDto).orElse(null);
		
	}

	/**
	 * Create a new note
	 *
	 * @param noteDto
	 * @return
	 */
	@RequestMapping(value = "/notes", method = RequestMethod.POST)
	public NoteDto create(@RequestBody NoteDto noteDto) {
		Note note = new Note();
		note.setId(null); // There should not be an id when creating a new record
		noteMapper.toEntity(noteDto, note); // The mapper will copy all values from the dto
		this.noteRepository.save(note);
		return noteMapper.toDto(note);
	}

	/**
	 * Update the note with the given id
	 *
	 * @param id
	 * @param noteDto
	 * @return
	 */
	@RequestMapping(value = "/notes/{id}", method = RequestMethod.PUT)
	public NoteDto update(@PathVariable("id") Long id, @RequestBody NoteDto noteDto) {
		// Retrieve the note by the id
		Optional<Note> optionalNote = this.noteRepository.findById(id);
		if (optionalNote.isPresent()) {
			Note note = optionalNote.get();
			noteMapper.toEntity(noteDto, note);
			this.noteRepository.save(note);
			return noteMapper.toDto(note);
		}

		return null;
	}

}
```

Now you can do this for groups.
