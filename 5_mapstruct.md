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
    <java.version>1.8</java.version>
    <org.mapstruct.version>1.2.0.Final</org.mapstruct.version>
    ...
</properties>
```

Next add the maven dependency for MapStruct.

```xml
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct-jdk8</artifactId>
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
    <version>3.5.1</version>
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
                <version>1.18.0</version>
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

}
```

That's it. This will create an mapper class which copies all the properties and will ignore the id when it goes back to an entity
You can even view the generated source in the target/generated-sources folder.

NoteMapperImpl.java
```java
package com.example.nextnote.note;

import javax.annotation.Generated;
import org.springframework.stereotype.Component;

@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2018-09-06T11:51:35+0200",
    comments = "version: 1.2.0.Final, compiler: javac, environment: Java 1.8.0_181 (Oracle Corporation)"
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
}
```

As you can see above the class is annotated with @Component. This allows us to use @AutoWired in spring.

Example: 

```java
// The preferred way is by constructor, but this is easier for the example
@AutoWired
private NoteMapper noteMapper;

public NoteDto doSomething(Note note) {
	NoteDto noteDto = this.noteMapper.toDto(note);
	// More code
	return noteDto;
}

```