# Lab 5: Connect React to a Mock API

### Week 6 | Frontend-Backend I — APIs

## Overview

In this lab, you'll connect a React frontend to a mock REST API using the Fetch API. You'll build a **book collection manager** that performs all four CRUD operations (Create, Read, Update, Delete) against a local JSON server. Along the way, you'll handle loading states, errors, and HTTP status codes—putting this week's readings into practice.

By the end of this lab, you'll understand how frontend applications communicate with backend services through HTTP, and you'll have experience managing the asynchronous nature of network requests in React.

**Prerequisites:**
- Completed Labs 1–4
- Week 7 readings completed
- Node.js 20+ installed
- Familiarity with React components and hooks (from Labs 3–4)

---

> [!IMPORTANT]
> **Windows Users:** We recommend using PowerShell rather than Command Prompt. Where commands differ between operating systems, both versions are provided. PowerShell commands are compatible with the Linux/macOS versions in most cases.

---

## Learning Objectives

By the end of this lab, you will be able to:

1. **Configure** a local mock REST API using [json-server](https://github.com/typicode/json-server)
2. **Implement** fetch requests using appropriate HTTP methods (GET, POST, PUT, DELETE)
3. **Handle** asynchronous state in React (loading, error, and success states)
4. **Interpret** HTTP status codes and respond to error conditions gracefully
5. **Apply** RESTful conventions when structuring API calls
6. **Test** API integration logic with mocked fetch calls

---

## Connection to Readings

This lab directly applies concepts from your Week 7 readings:

### From "HTTP Request Methods" (MDN)
- **Method semantics:** The MDN reference defines when to use GET, POST, PUT, PATCH, and DELETE. In this lab, you'll use all of these methods to manage your book collection—GET to list and retrieve books, POST to add new ones, PUT to update existing entries, and DELETE to remove them.
- **Idempotency:** MDN explains which methods are idempotent (safe to repeat). You'll see this in practice—calling GET or PUT multiple times produces the same result, while POST creates a new resource each time.

### From "REST" (MDN Glossary)
- **REST constraints:** The glossary defines REST as an architectural style with specific constraints including a uniform interface and stateless communication. Your mock API follows these constraints—each request contains all information the server needs, and resources are identified by URLs.
- **Resource-based URLs:** You'll work with endpoints like `/books` and `/books/:id`, following the resource-oriented URL patterns described in the glossary.

### From "Understanding And Using REST APIs" (Smashing Magazine)
- **Endpoint structure:** The Smashing Magazine article walks through how REST APIs organize endpoints. You'll build a client that mirrors these patterns—`GET /books` for the collection, `GET /books/1` for a single resource, `POST /books` to create, and so on.
- **Headers and content types:** The article explains the role of headers like `Content-Type: application/json`. You'll set these headers on every request that sends a body (POST, PUT).
- **Error handling:** The article discusses how APIs communicate errors through status codes. You'll handle these in your React components, showing appropriate messages for 404s, 500s, and network failures.

---

## Getting Started

### Step 1: Clone Your Repository

```bash
git clone https://github.com/ClarkCollege-CSE-SoftwareEngineering/lab-5-book-api-YOURUSERNAME.git
cd lab-5-book-api-YOURUSERNAME
```

### Step 2: Install Dependencies

```bash
npm install
```

### Step 3: Verify the Setup

Run the tests to confirm the starter configuration works:

```bash
npm test
```

Press `q` to exit watch mode, then verify TypeScript compiles:

```bash
npm run typecheck
```

### Step 4: Start the Mock API Server

Open a **separate terminal** and run:

```bash
npm run server
```

You should see output like:

```
  \{^_^}/ hi!

  Loading db.json
  Done

  Resources
  http://localhost:3001/books

  Home
  http://localhost:3001
```

✅ **Checkpoint:** Open `http://localhost:3001/books` in your browser. You should see the JSON array of three books.

🤔 **Reflection Question:** [json-server](https://github.com/typicode/json-server) automatically provides GET, POST, PUT, PATCH, and DELETE endpoints for each resource in `db.json`. How does this relate to the REST architectural constraints you read about in the MDN Glossary—particularly the concept of a uniform interface?

---

## Part 1: Building the API Client (20 minutes)

Now let's create a typed API client that communicates with our mock server.

### Step 1.1: Define the Book Type and API Module

Create `src/api/bookApi.ts`:

```typescript
export interface Book {
  id: number;
  title: string;
  author: string;
  year: number;
  genre: string;
}

export type NewBook = Omit<Book, 'id'>;

const API_BASE = 'http://localhost:3001/books';

export async function fetchBooks(): Promise<Book[]> {
  const response = await fetch(API_BASE);

  if (!response.ok) {
    throw new Error(`Failed to fetch books: ${response.status} ${response.statusText}`);
  }

  return response.json();
}

export async function fetchBookById(id: number): Promise<Book> {
  const response = await fetch(`${API_BASE}/${id}`);

  if (!response.ok) {
    if (response.status === 404) {
      throw new Error(`Book with id ${id} not found`);
    }
    throw new Error(`Failed to fetch book: ${response.status} ${response.statusText}`);
  }

  return response.json();
}

export async function createBook(book: NewBook): Promise<Book> {
  const response = await fetch(API_BASE, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(book),
  });

  if (!response.ok) {
    throw new Error(`Failed to create book: ${response.status} ${response.statusText}`);
  }

  return response.json();
}

export async function updateBook(id: number, book: Partial<NewBook>): Promise<Book> {
  const response = await fetch(`${API_BASE}/${id}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(book),
  });

  if (!response.ok) {
    if (response.status === 404) {
      throw new Error(`Book with id ${id} not found`);
    }
    throw new Error(`Failed to update book: ${response.status} ${response.statusText}`);
  }

  return response.json();
}

export async function deleteBook(id: number): Promise<void> {
  const response = await fetch(`${API_BASE}/${id}`, {
    method: 'DELETE',
  });

  if (!response.ok) {
    if (response.status === 404) {
      throw new Error(`Book with id ${id} not found`);
    }
    throw new Error(`Failed to delete book: ${response.status} ${response.statusText}`);
  }
}
```

✅ **Checkpoint:** Run `npm run typecheck` — it should pass with no errors.

🤔 **Reflection Question:** Notice that every function checks `response.ok` before parsing the body. The Fetch API only rejects for network failures, not HTTP error status codes. How does this relate to what the MDN HTTP Methods documentation says about the difference between a successful HTTP transaction and a successful application response?

---

## Part 2: React Components (25 minutes)

Now let's build the React components that display and manage books.

### Step 2.1: Create the BookCard Component

Create `src/components/BookCard.tsx`:

```tsx
import React from 'react';
import { Book } from '../api/bookApi';

interface BookCardProps {
  book: Book;
  onEdit: (book: Book) => void;
  onDelete: (id: number) => void;
}

export function BookCard({ book, onEdit, onDelete }: BookCardProps) {
  return (
    <article aria-label={book.title}>
      <h2>{book.title}</h2>
      <p>
        by {book.author} ({book.year})
      </p>
      <p>
        <em>{book.genre}</em>
      </p>
      <div>
        <button
          onClick={() => onEdit(book)}
          aria-label={`Edit ${book.title}`}
        >
          Edit
        </button>
        <button
          onClick={() => onDelete(book.id)}
          aria-label={`Delete ${book.title}`}
        >
          Delete
        </button>
      </div>
    </article>
  );
}
```

### Step 2.2: Create the BookForm Component

Create `src/components/BookForm.tsx`:

```tsx
import React, { useState, useEffect } from 'react';
import { Book, NewBook } from '../api/bookApi';

interface BookFormProps {
  onSubmit: (book: NewBook) => void;
  editingBook?: Book | null;
  onCancelEdit?: () => void;
}

export function BookForm({ onSubmit, editingBook, onCancelEdit }: BookFormProps) {
  const [title, setTitle] = useState('');
  const [author, setAuthor] = useState('');
  const [year, setYear] = useState('');
  const [genre, setGenre] = useState('');

  useEffect(() => {
    if (editingBook) {
      setTitle(editingBook.title);
      setAuthor(editingBook.author);
      setYear(String(editingBook.year));
      setGenre(editingBook.genre);
    } else {
      setTitle('');
      setAuthor('');
      setYear('');
      setGenre('');
    }
  }, [editingBook]);

  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();

    const trimmedTitle = title.trim();
    const trimmedAuthor = author.trim();

    if (!trimmedTitle || !trimmedAuthor) return;

    onSubmit({
      title: trimmedTitle,
      author: trimmedAuthor,
      year: parseInt(year, 10) || new Date().getFullYear(),
      genre: genre.trim() || 'Uncategorized',
    });

    if (!editingBook) {
      setTitle('');
      setAuthor('');
      setYear('');
      setGenre('');
    }
  }

  function handleCancel() {
    setTitle('');
    setAuthor('');
    setYear('');
    setGenre('');
    onCancelEdit?.();
  }

  return (
    <form onSubmit={handleSubmit} aria-label={editingBook ? 'Edit book' : 'Add new book'}>
      <div>
        <label htmlFor="title">Title</label>
        <input
          id="title"
          type="text"
          value={title}
          onChange={(e) => setTitle(e.target.value)}
          required
        />
      </div>
      <div>
        <label htmlFor="author">Author</label>
        <input
          id="author"
          type="text"
          value={author}
          onChange={(e) => setAuthor(e.target.value)}
          required
        />
      </div>
      <div>
        <label htmlFor="year">Year</label>
        <input
          id="year"
          type="number"
          value={year}
          onChange={(e) => setYear(e.target.value)}
        />
      </div>
      <div>
        <label htmlFor="genre">Genre</label>
        <input
          id="genre"
          type="text"
          value={genre}
          onChange={(e) => setGenre(e.target.value)}
        />
      </div>
      <div>
        <button type="submit">
          {editingBook ? 'Save Changes' : 'Add Book'}
        </button>
        {editingBook && (
          <button
            type="button"
            onClick={handleCancel}
            aria-label="Cancel editing"
          >
            Cancel
          </button>
        )}
      </div>
    </form>
  );
}
```

### Step 2.3: Create the BookList Component (Main Page)

Create `src/components/BookList.tsx`:

```tsx
import React, { useState, useEffect } from 'react';
import { BookCard } from './BookCard';
import { BookForm } from './BookForm';
import * as bookApi from '../api/bookApi';
import { Book, NewBook } from '../api/bookApi';

export function BookList() {
  const [books, setBooks] = useState<Book[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const [editingBook, setEditingBook] = useState<Book | null>(null);

  useEffect(() => {
    loadBooks();
  }, []);

  async function loadBooks() {
    try {
      setIsLoading(true);
      setError(null);
      const fetchedBooks = await bookApi.fetchBooks();
      setBooks(fetchedBooks);
    } catch (err) {
      setError('Failed to load books. Is the API server running?');
    } finally {
      setIsLoading(false);
    }
  }

  async function handleAdd(newBook: NewBook) {
    try {
      setError(null);
      const created = await bookApi.createBook(newBook);
      setBooks((prev) => [...prev, created]);
    } catch (err) {
      setError('Failed to add book. Please try again.');
    }
  }

  async function handleUpdate(newBook: NewBook) {
    if (!editingBook) return;

    try {
      setError(null);
      const updated = await bookApi.updateBook(editingBook.id, newBook);
      setBooks((prev) =>
        prev.map((b) => (b.id === editingBook.id ? updated : b))
      );
      setEditingBook(null);
    } catch (err) {
      setError('Failed to update book. Please try again.');
    }
  }

  async function handleDelete(id: number) {
    try {
      setError(null);
      await bookApi.deleteBook(id);
      setBooks((prev) => prev.filter((b) => b.id !== id));
      if (editingBook?.id === id) {
        setEditingBook(null);
      }
    } catch (err) {
      setError('Failed to delete book. Please try again.');
    }
  }

  if (isLoading) {
    return <p role="status">Loading books...</p>;
  }

  return (
    <div>
      <h1>Book Collection</h1>

      {error && (
        <p role="alert" style={{ color: '#dc3545' }}>
          {error}
        </p>
      )}

      <BookForm
        onSubmit={editingBook ? handleUpdate : handleAdd}
        editingBook={editingBook}
        onCancelEdit={() => setEditingBook(null)}
      />

      {books.length === 0 ? (
        <p>No books yet. Add one above!</p>
      ) : (
        <section aria-label="Book list">
          {books.map((book) => (
            <BookCard
              key={book.id}
              book={book}
              onEdit={setEditingBook}
              onDelete={handleDelete}
            />
          ))}
        </section>
      )}
    </div>
  );
}
```

✅ **Checkpoint:** Run `npm run typecheck` — it should pass with no errors.

---

## Part 3: Testing the API Client (20 minutes)

Now let's test the API client by mocking `global.fetch`.

### Step 3.1: Create API Client Tests

Create `src/__tests__/bookApi.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { fetchBooks, fetchBookById, createBook, deleteBook, updateBook } from '../api/bookApi';

describe('bookApi', () => {
  const originalFetch = global.fetch;

  beforeEach(() => {
    global.fetch = vi.fn();
  });

  afterEach(() => {
    global.fetch = originalFetch;
  });

  describe('fetchBooks', () => {
    it('returns books on successful response', async () => {
      const mockBooks = [
        { id: 1, title: 'Test Book', author: 'Author', year: 2024, genre: 'Fiction' },
      ];

      vi.mocked(global.fetch).mockResolvedValue({
        ok: true,
        json: () => Promise.resolve(mockBooks),
      } as Response);

      const result = await fetchBooks();

      expect(result).toEqual(mockBooks);
      expect(global.fetch).toHaveBeenCalledWith('http://localhost:3001/books');
    });

    it('throws error on failed response', async () => {
      vi.mocked(global.fetch).mockResolvedValue({
        ok: false,
        status: 500,
        statusText: 'Internal Server Error',
      } as Response);

      await expect(fetchBooks()).rejects.toThrow('Failed to fetch books');
    });
  });

  describe('fetchBookById', () => {
    it('returns a single book on success', async () => {
      const mockBook = { id: 1, title: 'Test', author: 'Author', year: 2024, genre: 'Fiction' };

      vi.mocked(global.fetch).mockResolvedValue({
        ok: true,
        json: () => Promise.resolve(mockBook),
      } as Response);

      const result = await fetchBookById(1);

      expect(result).toEqual(mockBook);
      expect(global.fetch).toHaveBeenCalledWith('http://localhost:3001/books/1');
    });

    it('throws specific error for 404 response', async () => {
      vi.mocked(global.fetch).mockResolvedValue({
        ok: false,
        status: 404,
        statusText: 'Not Found',
      } as Response);

      await expect(fetchBookById(99)).rejects.toThrow('Book with id 99 not found');
    });
  });

  describe('createBook', () => {
    it('sends POST request with correct headers and body', async () => {
      const newBook = { title: 'New Book', author: 'New Author', year: 2025, genre: 'Sci-Fi' };
      const createdBook = { id: 4, ...newBook };

      vi.mocked(global.fetch).mockResolvedValue({
        ok: true,
        json: () => Promise.resolve(createdBook),
      } as Response);

      const result = await createBook(newBook);

      expect(result).toEqual(createdBook);
      expect(global.fetch).toHaveBeenCalledWith('http://localhost:3001/books', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(newBook),
      });
    });

    // TODO: Add test for createBook error handling
  });

  describe('updateBook', () => {
    it('sends PUT request with correct method and body', async () => {
      const updatedData = { title: 'Updated Title', author: 'Author', year: 2024, genre: 'Fiction' };
      const updatedBook = { id: 1, ...updatedData };

      vi.mocked(global.fetch).mockResolvedValue({
        ok: true,
        json: () => Promise.resolve(updatedBook),
      } as Response);

      const result = await updateBook(1, updatedData);

      expect(result).toEqual(updatedBook);
      expect(global.fetch).toHaveBeenCalledWith('http://localhost:3001/books/1', {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(updatedData),
      });
    });

    // TODO: Add test for updateBook 404 error handling
  });

  describe('deleteBook', () => {
    it('sends DELETE request to correct endpoint', async () => {
      vi.mocked(global.fetch).mockResolvedValue({
        ok: true,
        json: () => Promise.resolve({}),
      } as Response);

      await deleteBook(1);

      expect(global.fetch).toHaveBeenCalledWith('http://localhost:3001/books/1', {
        method: 'DELETE',
      });
    });

    // TODO: Add test for deleteBook error handling
    // TODO: Add test for deleteBook 404 error handling
  });
});
```

✅ **Checkpoint:** Run `npm test` — all API tests should pass.

🤔 **Reflection Question:** We mock `global.fetch` rather than hitting the real server. What are the trade-offs of this approach compared to running tests against the actual json-server? Think about speed, reliability, and what each approach actually verifies.

---

## Part 4: Testing React Components (25 minutes)

Now let's test the React components with mocked API calls.

### Step 4.1: Create BookCard Tests

Create `src/__tests__/BookCard.test.tsx`:

```tsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { BookCard } from '../components/BookCard';
import { Book } from '../api/bookApi';

const mockBook: Book = {
  id: 1,
  title: 'The Pragmatic Programmer',
  author: 'David Thomas & Andrew Hunt',
  year: 2019,
  genre: 'Software Engineering',
};

describe('BookCard', () => {
  it('renders book title, author, year, and genre', () => {
    render(<BookCard book={mockBook} onEdit={vi.fn()} onDelete={vi.fn()} />);

    expect(screen.getByRole('heading', { name: /pragmatic programmer/i })).toBeInTheDocument();
    expect(screen.getByText(/david thomas & andrew hunt/i)).toBeInTheDocument();
    expect(screen.getByText(/2019/)).toBeInTheDocument();
    expect(screen.getByText(/software engineering/i)).toBeInTheDocument();
  });

  it('renders as an article with accessible name', () => {
    render(<BookCard book={mockBook} onEdit={vi.fn()} onDelete={vi.fn()} />);

    expect(screen.getByRole('article', { name: /pragmatic programmer/i })).toBeInTheDocument();
  });

  it('calls onEdit with the book when edit button is clicked', async () => {
    const user = userEvent.setup();
    const onEdit = vi.fn();

    render(<BookCard book={mockBook} onEdit={onEdit} onDelete={vi.fn()} />);

    await user.click(screen.getByRole('button', { name: /edit/i }));

    expect(onEdit).toHaveBeenCalledWith(mockBook);
  });

  it('calls onDelete with the book id when delete button is clicked', async () => {
    const user = userEvent.setup();
    const onDelete = vi.fn();

    render(<BookCard book={mockBook} onEdit={vi.fn()} onDelete={onDelete} />);

    await user.click(screen.getByRole('button', { name: /delete/i }));

    expect(onDelete).toHaveBeenCalledWith(1);
  });

  // TODO: Add a test that verifies the edit button has an accessible label
  // including the book title (e.g., "Edit The Pragmatic Programmer")
});
```

### Step 4.2: Create BookForm Tests

Create `src/__tests__/BookForm.test.tsx`:

```tsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { BookForm } from '../components/BookForm';

describe('BookForm', () => {
  it('renders all form fields with labels', () => {
    render(<BookForm onSubmit={vi.fn()} />);

    expect(screen.getByLabelText(/title/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/author/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/year/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/genre/i)).toBeInTheDocument();
  });

  it('submits form data when all required fields are filled', async () => {
    const user = userEvent.setup();
    const onSubmit = vi.fn();

    render(<BookForm onSubmit={onSubmit} />);

    await user.type(screen.getByLabelText(/title/i), 'New Book');
    await user.type(screen.getByLabelText(/author/i), 'Test Author');
    await user.type(screen.getByLabelText(/year/i), '2025');
    await user.type(screen.getByLabelText(/genre/i), 'Fiction');
    await user.click(screen.getByRole('button', { name: /add book/i }));

    expect(onSubmit).toHaveBeenCalledWith({
      title: 'New Book',
      author: 'Test Author',
      year: 2025,
      genre: 'Fiction',
    });
  });

  it('clears form after successful submission', async () => {
    const user = userEvent.setup();

    render(<BookForm onSubmit={vi.fn()} />);

    await user.type(screen.getByLabelText(/title/i), 'New Book');
    await user.type(screen.getByLabelText(/author/i), 'Test Author');
    await user.click(screen.getByRole('button', { name: /add book/i }));

    expect(screen.getByLabelText(/title/i)).toHaveValue('');
    expect(screen.getByLabelText(/author/i)).toHaveValue('');
  });

  it('does not submit when title is empty', async () => {
    const user = userEvent.setup();
    const onSubmit = vi.fn();

    render(<BookForm onSubmit={onSubmit} />);

    await user.type(screen.getByLabelText(/author/i), 'Test Author');
    await user.click(screen.getByRole('button', { name: /add book/i }));

    expect(onSubmit).not.toHaveBeenCalled();
  });

  it('populates form fields when editing a book', () => {
    const editBook = {
      id: 1,
      title: 'Existing Book',
      author: 'Existing Author',
      year: 2020,
      genre: 'Mystery',
    };

    render(<BookForm onSubmit={vi.fn()} editingBook={editBook} />);

    expect(screen.getByLabelText(/title/i)).toHaveValue('Existing Book');
    expect(screen.getByLabelText(/author/i)).toHaveValue('Existing Author');
    expect(screen.getByRole('button', { name: /save changes/i })).toBeInTheDocument();
  });

  it('shows cancel button when editing and calls onCancelEdit', async () => {
    const user = userEvent.setup();
    const onCancelEdit = vi.fn();
    const editBook = {
      id: 1,
      title: 'Book',
      author: 'Author',
      year: 2020,
      genre: 'Fiction',
    };

    render(
      <BookForm onSubmit={vi.fn()} editingBook={editBook} onCancelEdit={onCancelEdit} />
    );

    await user.click(screen.getByRole('button', { name: /cancel/i }));

    expect(onCancelEdit).toHaveBeenCalled();
  });

  // TODO: Add a test that verifies the form's accessible label changes
  // between "Add new book" and "Edit book" based on the editingBook prop
});
```

### Step 4.3: Create BookList Tests with Mocked API

Create `src/__tests__/BookList.test.tsx`:

```tsx
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { BookList } from '../components/BookList';
import * as bookApi from '../api/bookApi';

// Mock the entire API module
vi.mock('../api/bookApi');

// Type the mocked module
const mockedBookApi = vi.mocked(bookApi);

describe('BookList', () => {
  beforeEach(() => {
    vi.resetAllMocks();
  });

  describe('loading state', () => {
    it('shows loading message while fetching books', () => {
      mockedBookApi.fetchBooks.mockImplementation(
        () => new Promise(() => {})
      );

      render(<BookList />);

      expect(screen.getByRole('status')).toHaveTextContent(/loading/i);
    });
  });

  describe('displaying books', () => {
    it('renders fetched books', async () => {
      mockedBookApi.fetchBooks.mockResolvedValue([
        { id: 1, title: 'Book One', author: 'Author A', year: 2020, genre: 'Fiction' },
        { id: 2, title: 'Book Two', author: 'Author B', year: 2021, genre: 'Non-Fiction' },
      ]);

      render(<BookList />);

      expect(await screen.findByText('Book One')).toBeInTheDocument();
      expect(screen.getByText('Book Two')).toBeInTheDocument();
    });

    it('shows empty message when no books exist', async () => {
      mockedBookApi.fetchBooks.mockResolvedValue([]);

      render(<BookList />);

      expect(await screen.findByText(/no books yet/i)).toBeInTheDocument();
    });
  });

  describe('error handling', () => {
    it('shows error message when fetch fails', async () => {
      mockedBookApi.fetchBooks.mockRejectedValue(new Error('Network error'));

      render(<BookList />);

      expect(await screen.findByRole('alert')).toHaveTextContent(/failed to load/i);
    });
  });

  describe('adding books', () => {
    it('adds a new book when form is submitted', async () => {
      const user = userEvent.setup();

      mockedBookApi.fetchBooks.mockResolvedValue([]);
      mockedBookApi.createBook.mockResolvedValue({
        id: 1,
        title: 'New Book',
        author: 'New Author',
        year: 2025,
        genre: 'Fiction',
      });

      render(<BookList />);

      await screen.findByText(/no books yet/i);

      await user.type(screen.getByLabelText(/title/i), 'New Book');
      await user.type(screen.getByLabelText(/author/i), 'New Author');
      await user.type(screen.getByLabelText(/year/i), '2025');
      await user.type(screen.getByLabelText(/genre/i), 'Fiction');
      await user.click(screen.getByRole('button', { name: /add book/i }));

      expect(mockedBookApi.createBook).toHaveBeenCalledWith({
        title: 'New Book',
        author: 'New Author',
        year: 2025,
        genre: 'Fiction',
      });

      expect(await screen.findByText('New Book')).toBeInTheDocument();
    });
  });

  describe('deleting books', () => {
    it('removes a book when delete button is clicked', async () => {
      const user = userEvent.setup();

      mockedBookApi.fetchBooks.mockResolvedValue([
        { id: 1, title: 'Book to Delete', author: 'Author', year: 2024, genre: 'Fiction' },
      ]);
      mockedBookApi.deleteBook.mockResolvedValue();

      render(<BookList />);

      await screen.findByText('Book to Delete');

      await user.click(screen.getByRole('button', { name: /delete/i }));

      expect(mockedBookApi.deleteBook).toHaveBeenCalledWith(1);

      await waitFor(() => {
        expect(screen.queryByText('Book to Delete')).not.toBeInTheDocument();
      });
    });
  });

  describe('editing books', () => {
    it('populates form when edit is clicked and updates on save', async () => {
      const user = userEvent.setup();

      mockedBookApi.fetchBooks.mockResolvedValue([
        { id: 1, title: 'Original Title', author: 'Author', year: 2024, genre: 'Fiction' },
      ]);
      mockedBookApi.updateBook.mockResolvedValue({
        id: 1,
        title: 'Updated Title',
        author: 'Author',
        year: 2024,
        genre: 'Fiction',
      });

      render(<BookList />);

      await screen.findByText('Original Title');

      await user.click(screen.getByRole('button', { name: /edit/i }));

      const titleInput = screen.getByLabelText(/title/i);
      await user.clear(titleInput);
      await user.type(titleInput, 'Updated Title');

      await user.click(screen.getByRole('button', { name: /save changes/i }));

      expect(mockedBookApi.updateBook).toHaveBeenCalledWith(1, expect.objectContaining({
        title: 'Updated Title',
      }));

      await waitFor(() => {
        expect(screen.getByText('Updated Title')).toBeInTheDocument();
      });
    });
  });

  // TODO: Add a test for error handling when createBook fails
  // Hint: Use mockedBookApi.createBook.mockRejectedValue and verify the error alert appears

  // TODO: Add a test for error handling when deleteBook fails
  // Hint: Verify the book is NOT removed from the list when deletion fails
});
```

✅ **Checkpoint:** Run `npm test` — all tests should pass.

🤔 **Reflection Question:** Compare how we used `screen.findByText` (returns a Promise, waits for element) versus `screen.getByText` (synchronous, throws immediately if not found). When should you use each? Why is `findBy` essential when testing components that make API calls?

---

## Part 5: Your Turn — Write Your Own Tests (20 minutes)

Now it's time to apply what you've learned. Complete the TODO items in the test files above, plus add your own edge case tests.

### Task 5.1: Complete All TODOs

Find and implement all `TODO` comments in the test files:

1. **bookApi.test.ts:** Add tests for `createBook` error handling, `updateBook` 404 handling, `deleteBook` error handling, and `deleteBook` 404 handling
2. **BookCard.test.tsx:** Add a test verifying accessible button labels include the book title
3. **BookForm.test.tsx:** Add a test verifying the form's accessible label changes between add and edit modes
4. **BookList.test.tsx:** Add tests for `createBook` failure error handling and `deleteBook` failure error handling

### Task 5.2: Add Edge Case Tests

Add at least **2 more tests** to any of the test files that cover edge cases. Ideas:
- What happens when the form's author field is only whitespace?
- What happens when a book has a very long title?
- Test that the empty state returns after deleting the last book
- Test that the cancel button restores the "Add Book" label on the submit button

---

## Deliverables

Your submission should include:

```
book-api-lab/
├── src/
│   ├── components/
│   │   ├── BookCard.tsx
│   │   ├── BookForm.tsx
│   │   └── BookList.tsx
│   ├── api/
│   │   └── bookApi.ts
│   ├── __tests__/
│   │   ├── BookCard.test.tsx
│   │   ├── BookForm.test.tsx
│   │   ├── BookList.test.tsx
│   │   └── bookApi.test.ts
│   └── setupTests.ts
├── db.json
├── package.json
├── tsconfig.json
├── vitest.config.ts
└── README.md (your reflection)
```

### README.md Requirements

Your `README.md` must include:

1. **Your Name and Date**

2. **Reflection Section** (minimum 200 words) answering:
   - The Fetch API resolves its promise even for HTTP error status codes (like 404 or 500). How does checking `response.ok` address this? Why does the code throw different errors for different status codes?
   - Compare POST and PUT methods. When would you use each? Which is idempotent and why does that matter?
   - What are the trade-offs of mocking `global.fetch` versus running tests against the actual json-server?

3. **Key Concepts** section listing 3–5 concepts you learned

### Requirements Summary

- [ ] Minimum **30 passing tests**
- [ ] Minimum **90% code coverage**
- [ ] All TODO items completed
- [ ] README.md with reflection (200+ words) and key concepts
- [ ] TypeScript compiles without errors

---

## Grading Rubric

| Criteria | Points |
|----------|--------|
| Core functionality/fixes: API client implements all CRUD operations with correct HTTP methods, headers, and error handling | 30 |
| Student-added work: All TODOs completed + 2 edge case tests | 20 |
| Documentation deliverable: README reflection demonstrates understanding of REST concepts, `response.ok` behavior, and mocking trade-offs | 20 |
| README/reflection: Includes name, date, key concepts, meets 200-word minimum | 10 |
| Code quality: Clean TypeScript, semantic queries, proper async patterns | 10 |
| Quality metrics: 90%+ coverage, 30+ passing tests | 10 |
| **Total** | **100** |

---

## Stretch Goals

If you finish early, try these challenges:

1. **Add Search/Filter:** Implement a search bar that filters books by title or author. json-server supports query parameters like `?q=searchterm` and `?title_like=pattern`. Write tests for the filtering behavior.

2. **Add Pagination:** json-server supports `_page` and `_limit` query parameters. Implement "Load More" or page navigation and test the pagination logic.

3. **Use MSW:** Replace the `vi.mock` approach with [Mock Service Worker (MSW)](https://mswjs.io/) for more realistic API mocking that intercepts requests at the network level.

4. **Add Optimistic Updates:** Update the UI immediately when the user adds/deletes a book, then revert if the API call fails. Write tests for both the success and rollback scenarios.

---

## Troubleshooting

### json-server won't start

**Cause:** Version mismatch or port conflict.

**Solution:** The lab uses `json-server@0.17.4`. If you installed a different version:
```bash
npm uninstall json-server
npm install -D json-server@0.17.4
```

If port 3001 is in use, check what's using it:
```bash
# Linux/macOS/PowerShell:
lsof -i :3001

# Windows Command Prompt:
netstat -ano | findstr :3001
```

### Tests fail with "fetch is not defined"

**Cause:** Your test environment may not have `fetch` available.

**Solution:** The jsdom environment in Vitest includes `fetch` by default in Node 20+. Make sure you're running Node 20 or later:
```bash
node --version
```

### Tests reference localhost:3001 and fail in CI

**Cause:** Tests should never hit the real server. All API calls must be mocked.

**Solution:** Verify that all test files either:
1. Mock the entire API module with `vi.mock('../api/bookApi')`
2. Mock `global.fetch` with `vi.fn()`

GitHub Actions does not run json-server, so any test that hits the real endpoint will fail.

### Coverage is below 90%

**Cause:** Untested code paths.

**Solution:** Open `coverage/index.html` to see uncovered lines. Common misses:
- Error branches in API functions (404 vs. other errors)
- The `editingBook` code path in BookForm
- Cancel edit functionality
- Empty state rendering

### "Cannot find module" errors

**Cause:** Missing dependencies or incorrect import paths.

**Solution:**
1. Run `npm install` to ensure all dependencies are present
2. Check that import paths match file locations exactly (case-sensitive on Linux)
3. Verify `tsconfig.json` includes `"jsx": "react-jsx"`

---

## Submission

1. Push your code to your GitHub repository
2. Verify GitHub Actions passes all checks
3. Submit your repository URL via Canvas

**Due:** Tuesday, February 17, 2026 at 11:59 PM

> **Important:** The repository stops accepting commits at the deadline. Make sure to push your final work before then.
