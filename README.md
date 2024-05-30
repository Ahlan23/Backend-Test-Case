# Backend Test Case
## 1. Setup the NestJS Project
Install NestJS CLI and create a new project:
```
npm i -g @nestjs/cli
nest new library-management
cd library-management
```
## 2. Define Entities and DTOs
Create entities for **'Member'** and **'Book'**

**src/entities/book.entity.ts**
```
import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
export class Book {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  code: string;

  @Column()
  title: string;

  @Column()
  author: string;

  @Column()
  stock: number;
}
```
**src/entities/member.entity.ts**
```
import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
export class Member {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  code: string;

  @Column()
  name: string;

  @Column({ default: 0 })
  borrowedBooksCount: number;

  @Column({ default: false })
  isPenalized: boolean;

  @Column({ type: 'date', nullable: true })
  penaltyEndDate: Date;
}
```
Create DTOs for the requests and responses.

**src/dto/create-book.dto.ts**
```
export class CreateBookDto {
  readonly code: string;
  readonly title: string;
  readonly author: string;
  readonly stock: number;
}
export class CreateBookDto {
  readonly code: string;
  readonly title: string;
  readonly author: string;
  readonly stock: number;
}
```
**src/dto/create-member.dto.ts**
```
export class CreateMemberDto {
  readonly code: string;
  readonly name: string;
}
```
## 3. Create Services and Controllers
Create services for handling business logic.

**src/services/book.service.ts**
```
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Book } from '../entities/book.entity';
import { CreateBookDto } from '../dto/create-book.dto';

@Injectable()
export class BookService {
  constructor(
    @InjectRepository(Book)
    private booksRepository: Repository<Book>,
  ) {}

  async create(createBookDto: CreateBookDto): Promise<Book> {
    const book = this.booksRepository.create(createBookDto);
    return this.booksRepository.save(book);
  }

  async findAll(): Promise<Book[]> {
    return this.booksRepository.find();
  }
}
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Book } from '../entities/book.entity';
import { CreateBookDto } from '../dto/create-book.dto';

@Injectable()
export class BookService {
  constructor(
    @InjectRepository(Book)
    private booksRepository: Repository<Book>,
  ) {}

  async create(createBookDto: CreateBookDto): Promise<Book> {
    const book = this.booksRepository.create(createBookDto);
    return this.booksRepository.save(book);
  }

  async findAll(): Promise<Book[]> {
    return this.booksRepository.find();
  }
}
```
**src/services/member.service.ts**
```
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Member } from '../entities/member.entity';
import { CreateMemberDto } from '../dto/create-member.dto';

@Injectable()
export class MemberService {
  constructor(
    @InjectRepository(Member)
    private membersRepository: Repository<Member>,
  ) {}

  async create(createMemberDto: CreateMemberDto): Promise<Member> {
    const member = this.membersRepository.create(createMemberDto);
    return this.membersRepository.save(member);
  }

  async findAll(): Promise<Member[]> {
    return this.membersRepository.find();
  }
}
```
Create controllers for handling HTTP requests.

**src/controllers/book.controller.ts**
```
import { Controller, Get, Post, Body } from '@nestjs/common';
import { BookService } from '../services/book.service';
import { CreateBookDto } from '../dto/create-book.dto';
import { Book } from '../entities/book.entity';

@Controller('books')
export class BookController {
  constructor(private readonly bookService: BookService) {}

  @Post()
  create(@Body() createBookDto: CreateBookDto): Promise<Book> {
    return this.bookService.create(createBookDto);
  }

  @Get()
  findAll(): Promise<Book[]> {
    return this.bookService.findAll();
  }
}
```
**src/controllers/member.controller.ts**
```
import { Controller, Get, Post, Body } from '@nestjs/common';
import { MemberService } from '../services/member.service';
import { CreateMemberDto } from '../dto/create-member.dto';
import { Member } from '../entities/member.entity';

@Controller('members')
export class MemberController {
  constructor(private readonly memberService: MemberService) {}

  @Post()
  create(@Body() createMemberDto: CreateMemberDto): Promise<Member> {
    return this.memberService.create(createMemberDto);
  }

  @Get()
  findAll(): Promise<Member[]> {
    return this.memberService.findAll();
  }
}
```
## 4. Implement the borrowing and returning logic in the member service
**src/services/member.service.ts (add methods)**
```
async borrowBook(memberCode: string, bookCode: string): Promise<void> {
  const member = await this.membersRepository.findOne({ code: memberCode });
  if (!member || member.borrowedBooksCount >= 2 || member.isPenalized) {
    throw new Error('Cannot borrow book');
  }

  const book = await this.booksRepository.findOne({ code: bookCode });
  if (!book || book.stock < 1) {
    throw new Error('Book not available');
  }

  member.borrowedBooksCount += 1;
  book.stock -= 1;

  await this.membersRepository.save(member);
  await this.booksRepository.save(book);
}
```
## 5. Setup Database and Migrations
Install TypeORM and configure the database connection.
```
npm install @nestjs/typeorm typeorm sqlite3
```
**src/app.module.ts**
```
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { Book } from './entities/book.entity';
import { Member } from './entities/member.entity';
import { BookService } from './services/book.service';
import { MemberService } from './services/member.service';
import { BookController } from './controllers/book.controller';
import { MemberController } from './controllers/member.controller';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'sqlite',
      database: 'library.db',
      entities: [Book, Member],
      synchronize: true,
    }),
    TypeOrmModule.forFeature([Book, Member]),
  ],
  controllers: [BookController, MemberController],
  providers: [BookService, MemberService],
})
export class AppModule {}
```
## 6. Add Swagger for API Documentation
Install and configure Swagger.
```
npm install @nestjs/swagger swagger-ui-express
```
**src/main.ts**
```
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const config = new DocumentBuilder()
    .setTitle('Library Management')
    .setDescription('The Library Management API description')
    .setVersion('1.0')
    .build();
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, document);

  await app.listen(3000);
}
bootstrap();
```
## 7. Implement Unit Testing
Implement unit tests for services.

**src/services/book.service.spec.ts**
```
import { Test, TestingModule } from '@nestjs/testing';
import { BookService } from './book.service';
import { getRepositoryToken } from '@nestjs/typeorm';
import { Book } from '../entities/book.entity';
import { Repository } from 'typeorm';

describe('BookService', () => {
  let service: BookService;
  let repo: Repository<Book>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        BookService,
        {
          provide: getRepositoryToken(Book),
          useClass: Repository,
        },
      ],
    }).compile();

    service = module.get<BookService>(BookService);
    repo = module.get<Repository<Book>>(getRepositoryToken(Book));
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });
});
