# cis655-final-project
Book Recommendation Generator using Google Cloud

# Activate Google Books API
1. In Google Cloud go to "API" > "Library" > search for "Books API" > enable
2. Navigate to "API" > "Credentials" > "Create Credentials" > name the key "books api key" > restrict key to "books api" > create key.
3. Save the books api key for later.

# Use Google Colab to Generate a Sample Dataset
1. Use the API key you just created to create a csv file of books. The code I used gathers information on books for Kindergarten through 8th grade. Note that there is a query limit of 40 queries per minute (I set a delay of 100 seconds between grades) and a limit of 1000 queries per day.
2. After running the code, download the csv to your computer

```
import requests
import csv
import time

API_KEY = "insert your books api key here"

GRADE_QUERIES = {
    "K": "kindergarten books",
    "1": "grade 1 books",
    "2": "grade 2 books",
    "3": "grade 3 books",
    "4": "grade 4 books",
    "5": "grade 5 books",
    "6": "grade 6 books",
    "7": "grade 7 books",
    "8": "grade 8 books"
}

def fetch_books(query, grade, max_results=40):
    url = f"https://www.googleapis.com/books/v1/volumes?q={query}&maxResults={max_results}&printType=books&langRestrict=en&key={API_KEY}"
    response = requests.get(url)
    books = response.json().get("items", [])
    book_data = []

    for book in books:
        info = book.get("volumeInfo", {})
        sale = book.get("saleInfo", {})
        access = book.get("accessInfo", {})

        title = info.get("title", "")
        authors = ", ".join(info.get("authors", []))
        publisher = info.get("publisher", "")
        published_date = info.get("publishedDate", "")
        description = info.get("description", "")
        page_count = info.get("pageCount", "")
        categories = ", ".join(info.get("categories", []))
        maturity_rating = info.get("maturityRating", "")
        language = info.get("language", "")
        reading_mode_text = info.get("readingModes", {}).get("text", "")
        reading_mode_image = info.get("readingModes", {}).get("image", "")

        image_small = info.get("imageLinks", {}).get("smallThumbnail", "")
        image_large = info.get("imageLinks", {}).get("thumbnail", "")

        # ISBNs
        isbn_10 = ""
        isbn_13 = ""
        for identifier in info.get("industryIdentifiers", []):
            if identifier["type"] == "ISBN_10":
                isbn_10 = identifier["identifier"]
            elif identifier["type"] == "ISBN_13":
                isbn_13 = identifier["identifier"]

        # Sale Info
        country = sale.get("country", "")
        list_price_amount = sale.get("listPrice", {}).get("amount", "")
        list_price_currency = sale.get("listPrice", {}).get("currencyCode", "")
        buy_link = sale.get("buyLink", "")

        # Access Info
        web_reader_link = access.get("webReaderLink", "")
        embeddable = access.get("embeddable", "")

        book_data.append([
            title, authors, publisher, published_date, description,
            isbn_10, isbn_13,
            reading_mode_text, reading_mode_image, page_count,
            categories, maturity_rating,
            image_small, image_large,
            language, country,
            list_price_amount, list_price_currency,
            buy_link, web_reader_link, embeddable,
            grade
        ])

    return book_data

def save_to_csv(data, filename="k8_books_40.csv"):
    headers = [
        "Title", "Authors", "Publisher", "Published Date", "Description",
        "ISBN_10", "ISBN_13",
        "ReadingMode_Text", "ReadingMode_Image", "PageCount",
        "Categories", "MaturityRating",
        "Image_Small", "Image_Large",
        "Language", "Sale_Country",
        "ListPrice_Amount", "ListPrice_Currency",
        "BuyLink", "WebReaderLink", "Embeddable",
        "Grade"
    ]
    with open(filename, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.writer(file)
        writer.writerow(headers)
        writer.writerows(data)

# Run the scraping process
all_books = []
for grade, query in GRADE_QUERIES.items():
    print(f" Fetching books for Grade {grade}...")
    books = fetch_books(query, grade)
    all_books.extend(books)
    time.sleep(100)  # To avoid rate limits

save_to_csv(all_books)
print("CSV saved as k8_books_40.csv")
```

# Upload the csv file to a Cloud Storage bucket 
1. "Cloud Storage" > "Buckets" > "Create Bucket" > name your bucket, set a region, i made mine private
2. click on your bucket > "Upload" > "select files" > select your csv file

# Create a SQL Instance
1. "SQL" > "Intances" > "Create Instance" > name your instance (ex. book-recs-db) and select cheapest options. I enabled private (create a vpc to go with it, name it books-vpc, save the password you create) and public connectivity

# Connect SQL Instance to your bucket and get your csv file
1. Open CloudShell Terminal

1. start CloudProxy, it should return "Listening on...". If you see this, keep this open and open another tab. You will do this anytime you want to connect the session.
```
./cloud-sql-proxy book-recommendations-456120:us-central1:book-recs-db
#you can copy your connection info by clicking on instance and finding "connection name"
```

2. connect to PostgreSQL
```
psql -h 127.0.0.1 -U postgres -p 5432
```

3. create a new database
```
CREATE DATABASE book_recommendations_db;
```

4. connect to the database
```
\c book_recommendations_db
```

5. create a table. You may need to do this line by line. Type each line and hit enter. When finished, close the parentheses (see below) and hit enter.
```
CREATE TABLE books (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255),
    authors VARCHAR(500),
    publisher VARCHAR(255),
    published_date VARCHAR(20),
    description TEXT,
    isbn_10 VARCHAR(20),
    isbn_13 VARCHAR(20),
    reading_mode_text VARCHAR(50),
    reading_mode_image VARCHAR(50),
    page_count INTEGER,
    categories VARCHAR(500),
    maturity_rating VARCHAR(50),
    image_small VARCHAR(255),
    image_large VARCHAR(255),
    language VARCHAR(50),
    sale_country VARCHAR(50),
    list_price_amount DECIMAL(10, 2),
    list_price_currency VARCHAR(10),
    buy_link TEXT,
    web_reader_link TEXT,
    embeddable BOOLEAN,
    grade VARCHAR(10)
);
```

6. verify table creation
```
\dt
```

7.  import data from bucket to sql
```
gcloud sql import csv your-instance-name gs://your-bucket-name/k8_books_40.csv \
  --database=your-database-name \
  --table=books
```

8. create a directory
```
sudo mkdir /cloudsql
sudo chown $USER /cloudsql
```

9. re-run proxy
```
./cloud-sql-proxy \
  --unix-socket /cloudsql \
  [PROJECT_ID]:[REGION}:[INSTANCE-NAME]
```

10. reconnect to database (may new tab or terminal)
```
psql -h /cloudsql/[PROJECT_ID]:[REGION}:[INSTANCE-NAME] \
     -U postgres -d book_recommendations_db
```

11. copy csv and include headers; single line of code
```
\copy books(title,authors,publisher,published_date,description,isbn_10,isbn_13,reading_mode_text,reading_mode_image,page_count,categories,maturity_rating,image_small,image_large,language,sale_country,list_price_amount,list_price_currency,buy_link,web_reader_link,embeddable,grade) FROM 'k8_books_40.csv' WITH CSV HEADER;
```

# Create Backend API
1. Open Cloudshell terminal

2. create a folder and change to that directory
```
mkdir book-api && cd book-api
```

3. create a virtual environment
```
python3 -m venv venv
source venv/bin/activate
```

4. install dependencies
```
pip install flask psycopg2-binary flask_sqlalchemy python-dotenv fastapi uvicorn
```

5. create a .env file for your db (if this doesn't connect correctly, add to app.py code)
```
DB_USER=postgres
DB_PASSWORD=your_db_password (this is the password from when you made your private connection on your instance)
DB_NAME=book_recommendations_db
DB_HOST=127.0.0.1
```

6. check the variables are set, this should return the info you just entered in the last step
```
echo $DB_NAME
echo $DB_USER
echo $DB_PASSWORD
echo $DB_HOST
```
## `books` and app.py
7. click "Editor" button, navigate to books-api and create a new file named app.py
```
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import psycopg2
import os
from typing import List
from typing import List, Optional

app = FastAPI()

DB_NAME = os.getenv("DB_NAME")
DB_USER = os.getenv("DB_USER")
DB_PASSWORD = os.getenv("DB_PASSWORD")
DB_HOST = os.getenv("DB_HOST")

# Removed maturity_rating
class Book(BaseModel):
    title: Optional[str]
    authors: Optional[str]
    publisher: Optional[str]
    published_date: Optional[str]
    description: Optional[str]
    isbn_10: Optional[str]
    isbn_13: Optional[str]
    reading_mode_text: Optional[bool]
    reading_mode_image: Optional[bool]
    page_count: Optional[int]
    categories: Optional[str]
    image_small: Optional[str]
    image_large: Optional[str]
    language: Optional[str]
    sale_country: Optional[str]
    list_price_amount: Optional[float]
    list_price_currency: Optional[str]
    buy_link: Optional[str]
    web_reader_link: Optional[str]
    embeddable: Optional[bool]
    grade: Optional[str]

def get_db_connection():
    conn = psycopg2.connect(
        dbname=DB_NAME,
        user=DB_USER,
        password=DB_PASSWORD,
        host=DB_HOST
    )
    return conn

@app.get("/books", response_model=List[Book])
def get_books():
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        
        # Only select the columns that match the model
        cur.execute("""
            SELECT title, authors, publisher, published_date, description,
                   isbn_10, isbn_13, reading_mode_text, reading_mode_image,
                   page_count, categories, image_small, image_large, language,
                   sale_country, list_price_amount, list_price_currency,
                   buy_link, web_reader_link, embeddable, grade
            FROM books
        """)
        
        rows = cur.fetchall()
        columns = [desc[0] for desc in cur.description]
        cur.close()
        conn.close()
        return [dict(zip(columns, row)) for row in rows]
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

8. *run the cloud sql proxy, it should return "Listening on...". If you see this, keep this open and open another tab. You will do this anytime you want to connect the session.*
```
./cloud-sql-proxy [PROJECT-ID]:[REGION]:[INSTANCE-ID]
```

9. in the new tab, test your conneciton
```
psql -h /cloudsql/[PROJECT-ID]:[REGION]:[INSTANCE-ID] -U postgres -d book_recommendations_db
```

10. *ensure that you are in the right directory and activate the virtual environment (do this every new session)*
```
cd ~/book-api
source venv/bin/activate
```

11. Run your app  *(do this every time you change app.py or are starting a new session)*
```
uvicorn app:app --host=0.0.0.0 --port=8080 --reload
```

12. click the "Web Preview" button and select "preview on port 8080"
13. If this gives an error, click on the url and add /docs to the end and enter
    orginal url: "https://8080-cs-399637968661-default.cs-us-central1-pits.cloudshell.dev/?authuser=0"
    new url: "https://8080-cs-399637968661-default.cs-us-central1-pits.cloudshell.dev/docs"
14. Update app.py to add ability to search for books by title, author, category, and grade. Also add ability to add, update, and delete books.
```
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import psycopg2
import os
from typing import List
from typing import List, Optional

app = FastAPI()

#use these 4 if using .env file
DB_NAME = os.getenv("DB_NAME") 
DB_USER = os.getenv("DB_USER")
DB_PASSWORD = os.getenv("DB_PASSWORD")
DB_HOST = os.getenv("DB_HOST")

# Removed maturity_rating
class Book(BaseModel):
    title: Optional[str]
    authors: Optional[str]
    publisher: Optional[str]
    published_date: Optional[str]
    description: Optional[str]
    isbn_10: Optional[str]
    isbn_13: Optional[str]
    reading_mode_text: Optional[bool]
    reading_mode_image: Optional[bool]
    page_count: Optional[int]
    categories: Optional[str]
    image_small: Optional[str]
    image_large: Optional[str]
    language: Optional[str]
    sale_country: Optional[str]
    list_price_amount: Optional[float]
    list_price_currency: Optional[str]
    buy_link: Optional[str]
    web_reader_link: Optional[str]
    embeddable: Optional[bool]
    grade: Optional[str]

def get_db_connection(): # i entered mine cause it wasn't connecting, but can also use .env objects
    conn = psycopg2.connect(
        dbname="book_recommendations_db",
        user="postgres",
        password="stimac-cis655-final",
        host="127.0.0.1",
        port="5432"
    )
    return conn

from fastapi import Query

@app.get("/books", response_model=List[Book])
def get_books(
    title: str = Query(None),
    authors: str = Query(None),
    categories: str = Query(None),
    grade: str = Query(None)
):
    try:
        conn = get_db_connection()
        cur = conn.cursor()

        # Build query with filters
        query = """
            SELECT title, authors, publisher, published_date, description,
                   isbn_10, isbn_13, reading_mode_text, reading_mode_image,
                   page_count, categories, image_small, image_large, language,
                   sale_country, list_price_amount, list_price_currency,
                   buy_link, web_reader_link, embeddable, grade
            FROM books
        """
        filters = []
        values = []

        if title:
            filters.append("LOWER(title) LIKE %s")
            values.append(f"%{title.lower()}%")
        if authors:
            filters.append("LOWER(authors) LIKE %s")
            values.append(f"%{authors.lower()}%")
        if categories:
            filters.append("LOWER(categories) LIKE %s")
            values.append(f"%{categories.lower()}%")
        if grade:
            filters.append("LOWER(grade) = %s")
            values.append(grade.lower())

        if filters:
            query += " WHERE " + " AND ".join(filters)

        cur.execute(query, values)
        rows = cur.fetchall()
        columns = [desc[0] for desc in cur.description]
        cur.close()
        conn.close()

        return [dict(zip(columns, row)) for row in rows]

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

## new code below
# GET book by ISBN
@app.get("/books/isbn/{isbn}", response_model=Book)
def get_book_by_isbn(isbn: str):
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute("""
            SELECT title, authors, publisher, published_date, description,
                   isbn_10, isbn_13, reading_mode_text, reading_mode_image,
                   page_count, categories, image_small, image_large, language,
                   sale_country, list_price_amount, list_price_currency,
                   buy_link, web_reader_link, embeddable, grade
            FROM books
            WHERE isbn_13 = %s OR isbn_10 = %s
        """, (isbn, isbn))
        row = cur.fetchone()
        cur.close()
        conn.close()
        if row:
            columns = [desc[0] for desc in cur.description]
            return dict(zip(columns, row))
        else:
            raise HTTPException(status_code=404, detail="Book not found")
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))


from fastapi import HTTPException

# POST create a new book
@app.post("/books", response_model=Book)
def create_book(book: Book):
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        # Insert book into the database
        cur.execute("""
            INSERT INTO books (title, authors, publisher, published_date, description,
                               isbn_10, isbn_13, reading_mode_text, reading_mode_image,
                               page_count, categories, image_small, image_large, language,
                               sale_country, list_price_amount, list_price_currency,
                               buy_link, web_reader_link, embeddable, grade)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            RETURNING title, authors, publisher, published_date, description,
                       isbn_10, isbn_13, reading_mode_text, reading_mode_image,
                       page_count, categories, image_small, image_large, language,
                       sale_country, list_price_amount, list_price_currency,
                       buy_link, web_reader_link, embeddable, grade;
        """, (book.title, book.authors, book.publisher, book.published_date, book.description,
              book.isbn_10, book.isbn_13, book.reading_mode_text, book.reading_mode_image,
              book.page_count, book.categories, book.image_small, book.image_large, book.language,
              book.sale_country, book.list_price_amount, book.list_price_currency,
              book.buy_link, book.web_reader_link, book.embeddable, book.grade))
        
        # Commit changes and return the inserted data
        conn.commit()
        cur.close()
        conn.close()
        return book
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))


# PUT update a book by isbn
@app.put("/books/{isbn}", response_model=Book)
def update_book(isbn: str, book: Book):
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        # Update book in the database
        cur.execute("""
            UPDATE books
            SET title = %s, authors = %s, publisher = %s, published_date = %s, description = %s,
                isbn_10 = %s, isbn_13 = %s, reading_mode_text = %s, reading_mode_image = %s,
                page_count = %s, categories = %s, image_small = %s, image_large = %s, language = %s,
                sale_country = %s, list_price_amount = %s, list_price_currency = %s,
                buy_link = %s, web_reader_link = %s, embeddable = %s, grade = %s
            WHERE isbn_13 = %s OR isbn_10 = %s
            RETURNING title, authors, publisher, published_date, description,
                       isbn_10, isbn_13, reading_mode_text, reading_mode_image,
                       page_count, categories, image_small, image_large, language,
                       sale_country, list_price_amount, list_price_currency,
                       buy_link, web_reader_link, embeddable, grade;
        """, (book.title, book.authors, book.publisher, book.published_date, book.description,
              book.isbn_10, book.isbn_13, book.reading_mode_text, book.reading_mode_image,
              book.page_count, book.categories, book.image_small, book.image_large, book.language,
              book.sale_country, book.list_price_amount, book.list_price_currency,
              book.buy_link, book.web_reader_link, book.embeddable, book.grade,
              isbn, isbn))
        
        # Commit changes and return the updated data
        conn.commit()
        cur.close()
        conn.close()
        return book
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))


# DELETE a book by isbn
@app.delete("/books/{isbn}")
def delete_book(isbn: str):
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        # Delete book from the database
        cur.execute("""
            DELETE FROM books
            WHERE isbn_13 = %s OR isbn_10 = %s;
        """, (isbn, isbn))
        
        conn.commit()
        cur.close()
        conn.close()
        return {"message": "Book deleted successfully"}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```
*REMINDER*: Rerun uvicorn in terminal after changing app.py
```
uvicorn app:app --host=0.0.0.0 --port=8080 --reload
```


15. connect to database
```
psql "host=127.0.0.1 dbname=book_recommendations_db user=postgres password=stimac-cis655-final"
```
## `user` and `user_books`
16. create `user` and `user_books` table
```
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE user_books (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(user_id),
    title TEXT,
    author TEXT,
    rating INTEGER CHECK (rating >= 1 AND rating <= 5),
    read_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```
*REMINDER*: Rerun uvicorn in terminal after changing app.py
```
uvicorn app:app --host=0.0.0.0 --port=8080 --reload
```


17. insert some sample data
```
INSERT INTO users (username, email) VALUES
('alice', 'alice@example.com'),
('bob', 'bob@example.com');

INSERT INTO user_books (user_id, title, author, rating) VALUES
(1, 'Charlotte’s Web', 'E. B. White', 5),
(1, 'Matilda', 'Roald Dahl', 4),
(2, 'The Giver', 'Lois Lowry', 5),
(2, 'The Hobbit', 'J.R.R. Tolkien', 3);
```

18. update app.py for user and user_books
```
# USER options

# GET all users
@app.get("/users")
def get_users():
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users")
    users = cursor.fetchall()
    cursor.close()
    conn.close()
    return users

# GET user by ID
@app.get("/users/{user_id}")
def get_user(user_id: int):
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users WHERE user_id = %s", (user_id,))
    user = cursor.fetchone()
    cursor.close()
    conn.close()
    if user:
        return user
    raise HTTPException(status_code=404, detail="User not found")

# POST create new user
@app.post("/users")
def create_user(username: str, email: str):
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute(
        "INSERT INTO users (username, email) VALUES (%s, %s) RETURNING *",
        (username, email)
    )
    user = cursor.fetchone()
    conn.commit()
    cursor.close()
    conn.close()
    return user

# PUT update user
@app.put("/users/{user_id}")
def update_user(user_id: int, username: str, email: str):
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute(
        "UPDATE users SET username = %s, email = %s WHERE user_id = %s RETURNING *",
        (username, email, user_id)
    )
    user = cursor.fetchone()
    conn.commit()
    cursor.close()
    conn.close()
    if user:
        return user
    raise HTTPException(status_code=404, detail="User not found")

# DELETE user
@app.delete("/users/{user_id}")
def delete_user(user_id: int):
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("DELETE FROM users WHERE user_id = %s RETURNING *", (user_id,))
    user = cursor.fetchone()
    conn.commit()
    cursor.close()
    conn.close()
    if user:
        return {"message": "User deleted"}
    raise HTTPException(status_code=404, detail="User not found")


## USER_BOOKS 

class UserBook(BaseModel):
    user_id: int
    title: str
    author: str
    rating: Optional[int] = None
    read_at: Optional[str] = None  # or datetime if you prefer
    status: Optional[str] = None
    progress: Optional[int] = None
    notes: Optional[str] = None

# get all user-book enteries
@app.get("/user_books")
def get_all_user_books():
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM user_books")
    records = cursor.fetchall()
    cursor.close()
    conn.close()
    return records

# get books for specific user
from fastapi import Query

@app.get("/user_books/{user_id}")
def get_user_books(
    user_id: int,
    status: Optional[str] = Query(None),
    rating: Optional[int] = Query(None),
    progress: Optional[int] = Query(None)
):
    conn = get_db_connection()
    cursor = conn.cursor()

    query = "SELECT * FROM user_books WHERE user_id = %s"
    values = [user_id]

    if status:
        query += " AND status = %s"
        values.append(status)
    if rating:
        query += " AND rating = %s"
        values.append(rating)
    if progress:
        query += " AND progress = %s"
        values.append(progress)

    cursor.execute(query, values)
    user_books = cursor.fetchall()
    cursor.close()
    conn.close()
    return user_books


# create new user book entry
@app.post("/user_books")
def create_user_book(user_book: UserBook):
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("""
        INSERT INTO user_books (user_id, title, author, rating, status, progress, notes)
        VALUES (%s, %s, %s, %s, %s, %s, %s)
        RETURNING *;
    """, (
        user_book.user_id,
        user_book.title,
        user_book.author,
        user_book.rating,
        user_book.status,
        user_book.progress,
        user_book.notes
    ))
    new_entry = cursor.fetchone()
    conn.commit()
    cursor.close()
    conn.close()
    return new_entry

# update entry by id
@app.put("/user_books/{entry_id}")
def update_user_book(entry_id: int, user_book: UserBook):
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("""
        UPDATE user_books
        SET user_id = %s, title = %s, author = %s, rating = %s,
            status = %s, progress = %s, notes = %s
        WHERE id = %s
        RETURNING *;
    """, (
        user_book.user_id,
        user_book.title,
        user_book.author,
        user_book.rating,
        user_book.status,
        user_book.progress,
        user_book.notes,
        entry_id
    ))
    updated_entry = cursor.fetchone()
    conn.commit()
    cursor.close()
    conn.close()
    if updated_entry:
        return updated_entry
    raise HTTPException(status_code=404, detail="Entry not found")

# delete entry by id
@app.delete("/user_books/{entry_id}")
def delete_user_book(entry_id: int):
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("DELETE FROM user_books WHERE id = %s RETURNING *", (entry_id,))
    deleted = cursor.fetchone()
    conn.commit()
    cursor.close()
    conn.close()
    if deleted:
        return {"message": "User-book entry deleted"}
    raise HTTPException(status_code=404, detail="Entry not found")

```
*REMINDER*: Rerun uvicorn in terminal after changing app.py
```
uvicorn app:app --host=0.0.0.0 --port=8080 --reload
```



19. merge books and user_books
```
# make sure ./cloud-sql-proxy book-recommendations-456120:us-central1:book-recs-db is still running in other tab

# connect to the database
psql "host=127.0.0.1 dbname=book_recommendations_db user=postgres password=stimac-cis655-final"
```

20. update `user_books` (if needed)
```
ALTER TABLE user_books
ADD COLUMN status TEXT,
ADD COLUMN progress INTEGER,
ADD COLUMN notes TEXT,
ADD COLUMN isbn_10 TEXT;
```

21. In EDITOR, open the app.py
```
## USER_BOOKS 

class UserBook(BaseModel):
    user_id: int
    title: str
    author: str
    rating: Optional[int] = None
    read_at: Optional[str] = None  # or datetime if you prefer
    status: Optional[str] = None
    progress: Optional[int] = None
    notes: Optional[str] = None

# get all user-book enteries
@app.get("/user_books")
def get_all_user_books():
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM user_books")
    records = cursor.fetchall()
    cursor.close()
    conn.close()
    return records

# get books for specific user
from fastapi import Query

@app.get("/user_books/{user_id}")
def get_user_books(user_id: int, status: Optional[str] = Query(None)):
    try:
        conn = get_db_connection()
        cursor = conn.cursor()

        query = """
            SELECT b.title, b.authors, b.image_small, b.categories, b.grade,
                   ub.rating, ub.status, ub.progress, ub.notes, ub.read_at
            FROM user_books ub
            JOIN books b
              ON LOWER(ub.title) = LOWER(b.title)
             AND LOWER(ub.author) = LOWER(b.authors)
            WHERE ub.user_id = %s
        """
        params = [user_id]

        if status:
            query += " AND ub.status = %s"
            params.append(status)

        cursor.execute(query, params)
        rows = cursor.fetchall()
        columns = [desc[0] for desc in cursor.description]

        cursor.close()
        conn.close()

        return [dict(zip(columns, row)) for row in rows]

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))


# create new user book entry
@app.post("/user_books")
def create_user_book(user_book: UserBook):
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("""
        INSERT INTO user_books (user_id, title, author, rating, status, progress, notes)
        VALUES (%s, %s, %s, %s, %s, %s, %s)
        RETURNING *;
    """, (
        user_book.user_id,
        user_book.title,
        user_book.author,
        user_book.rating,
        user_book.status,
        user_book.progress,
        user_book.notes
    ))
    new_entry = cursor.fetchone()
    conn.commit()
    cursor.close()
    conn.close()
    return new_entry

# update entry by id
@app.put("/user_books/{entry_id}")
def update_user_book(entry_id: int, user_book: UserBook):
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("""
        UPDATE user_books
        SET user_id = %s, title = %s, author = %s, rating = %s,
            status = %s, progress = %s, notes = %s
        WHERE id = %s
        RETURNING *;
    """, (
        user_book.user_id,
        user_book.title,
        user_book.author,
        user_book.rating,
        user_book.status,
        user_book.progress,
        user_book.notes,
        entry_id
    ))
    updated_entry = cursor.fetchone()
    conn.commit()
    cursor.close()
    conn.close()
    if updated_entry:
        return updated_entry
    raise HTTPException(status_code=404, detail="Entry not found")

# delete entry by id
@app.delete("/user_books/{entry_id}")
def delete_user_book(entry_id: int):
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("DELETE FROM user_books WHERE id = %s RETURNING *", (entry_id,))
    deleted = cursor.fetchone()
    conn.commit()
    cursor.close()
    conn.close()
    if deleted:
        return {"message": "User-book entry deleted"}
    raise HTTPException(status_code=404, detail="Entry not found")


from fastapi import Query

@app.get("/user_books/{user_id}")
def get_user_books(user_id: int, status: Optional[str] = Query(None)):
    try:
        conn = get_db_connection()
        cursor = conn.cursor()

        query = """
            SELECT b.title, b.authors, b.image_small, b.categories, b.grade,
                   ub.rating, ub.status, ub.progress, ub.notes, ub.read_at
            FROM user_books ub
            JOIN books b ON ub.isbn_10 = b.isbn_10
            WHERE ub.user_id = %s
        """
        params = [user_id]

        if status:
            query += " AND ub.status = %s"
            params.append(status)

        cursor.execute(query, params)
        rows = cursor.fetchall()
        columns = [desc[0] for desc in cursor.description]

        cursor.close()
        conn.close()

        return [dict(zip(columns, row)) for row in rows]
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```
*REMINDER*: Rerun uvicorn in terminal after changing app.py
```
uvicorn app:app --host=0.0.0.0 --port=8080 --reload
```

22. return to the terminal and open your db again
```
psql "host=127.0.0.1 dbname=book_recommendations_db user=postgres password=stimac-cis655-final"
```

23. enable an extension to help with fuzzy matching, so that book titles and authors will match even if there are capitalization or minor spelling errors
```
CREATE EXTENSION IF NOT EXISTS pg_trgm;
``` 

24. open app.py in editor and update get_user_books
```
@app.get("/user_books/{user_id}")
def get_user_books(user_id: int, status: Optional[str] = Query(None)):
    try:
        conn = get_db_connection()
        cursor = conn.cursor()

        query = """
            SELECT b.title, b.authors, b.image_small, b.categories, b.grade,
                   ub.rating, ub.status, ub.progress, ub.notes, ub.read_at
            FROM user_books ub
            JOIN books b
              ON similarity(ub.title, b.title) > 0.4
             AND similarity(ub.author, b.authors) > 0.4
            WHERE ub.user_id = %s
        """
        params = [user_id]

        if status:
            query += " AND ub.status = %s"
            params.append(status)

        cursor.execute(query, params)
        rows = cursor.fetchall()
        columns = [desc[0] for desc in cursor.description]

        cursor.close()
        conn.close()

        return [dict(zip(columns, row)) for row in rows]

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

*REMINDER*: Rerun uvicorn in terminal after changing app.py
```
uvicorn app:app --host=0.0.0.0 --port=8080 --reload
```

26. update tables so that users can add books and ratings for books that arent already in the table. I did not include title and author in this update because we are keeping them as required.
```
sql "host=127.0.0.1 dbname=book_recommendations_db user=postgres password=stimac-cis655-final"

ALTER TABLE books
    ALTER COLUMN published_date   DROP NOT NULL,
    ALTER COLUMN description      DROP NOT NULL,
    ALTER COLUMN categories       DROP NOT NULL,
    ALTER COLUMN image_small      DROP NOT NULL,
    ALTER COLUMN image_large      DROP NOT NULL,
    ALTER COLUMN language         DROP NOT NULL,
    ALTER COLUMN sale_country     DROP NOT NULL,
    ALTER COLUMN grade            DROP NOT NULL;
```

27. update UserBook model in app.py
```
class UserBook(BaseModel):
    user_id: int
    title: str
    author: str
    isbn_10: Optional[str] = None  
    rating: Optional[int] = None
    read_at: Optional[str] = None
    status: Optional[str] = None
    progress: Optional[int] = None
    notes: Optional[str] = None
```


28. update POST/user_books model in app.py
```
@app.post("/user_books")
def add_user_book(entry: UserBook):
    conn = get_db_connection()
    cur  = conn.cursor()

    try:
        # 1. Does the book already exist?  (match by isbn_10 first, else title+author)
        cur.execute("""
            SELECT isbn_10 FROM books
            WHERE (isbn_10 IS NOT NULL AND isbn_10 = %s)
               OR (LOWER(title) = LOWER(%s) AND LOWER(authors) = LOWER(%s))
            LIMIT 1
        """, (entry.isbn_10, entry.title, entry.author))
        row = cur.fetchone()

        # 2. If missing, insert a minimal book row
        if not row:
            cur.execute("""
                INSERT INTO books (title, authors, isbn_10)
                VALUES (%s, %s, %s)
            """, (entry.title, entry.author, entry.isbn_10))

        # 3. Insert into user_books
        cur.execute("""
            INSERT INTO user_books (user_id, title, author, isbn_10,
                                    rating, status, progress, notes)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
            RETURNING *
        """, (
            entry.user_id, entry.title, entry.author, entry.isbn_10,
            entry.rating, entry.status, entry.progress, entry.notes
        ))
        new_row = cur.fetchone()
        conn.commit()
        return dict(zip([d[0] for d in cur.description], new_row))

    except Exception as e:
        conn.rollback()
        raise HTTPException(status_code=500, detail=str(e))
    finally:
        cur.close()
        conn.close()
```
*REMINDER*: Rerun uvicorn in terminal after changing app.py
```
uvicorn app:app --host=0.0.0.0 --port=8080 --reload
```

29. add a SQL-based recommendations line to app.py that will give recommendations based on top-rated, grade level books that are unread by the user. this is really more for testing.
```
@app.get("/recommendations/{user_id}")
def recommendations(user_id: int, limit: int = 5):
    conn = get_db_connection()
    cur  = conn.cursor()

    cur.execute("""
        WITH user_grade AS (
            SELECT DISTINCT b.grade
            FROM user_books ub
            JOIN books b ON ub.isbn_10 = b.isbn_10
            WHERE ub.user_id = %s
              AND ub.rating IS NOT NULL
        ),
        candidate_books AS (
            SELECT b.isbn_10, b.title, b.authors, b.grade,
                   COALESCE(AVG(ub2.rating),0) AS avg_rating,
                   COUNT(ub2.rating)           AS num_ratings
            FROM books b
            LEFT JOIN user_books ub2 ON b.isbn_10 = ub2.isbn_10
            WHERE b.grade IN (SELECT grade FROM user_grade)
              AND b.isbn_10 NOT IN (
                    SELECT isbn_10 FROM user_books WHERE user_id = %s
              )
            GROUP BY b.isbn_10, b.title, b.authors, b.grade
        )
        SELECT * FROM candidate_books
        ORDER BY avg_rating DESC, num_ratings DESC
        LIMIT %s;
    """, (user_id, user_id, limit))

    rows = cur.fetchall()
    cols = [d[0] for d in cur.description]
    cur.close(); conn.close()
    return [dict(zip(cols, r)) for r in rows]
```
*REMINDER*: Rerun uvicorn in terminal after changing app.py
```
uvicorn app:app --host=0.0.0.0 --port=8080 --reload
```

30. add another books csv to make the recommendations run better
```
# copy the csv from the bucket to the shell
gsutil cp gs://k8-books-bucket/books.csv .

# open the db
psql "host=127.0.0.1 dbname=book_recommendations_db user=postgres password=stimac-cis655-final"

# in the db, import the data from the new csv
\copy books(
    title,authors,publisher,published_date,description,
    isbn_10,isbn_13,reading_mode_text,reading_mode_image,
    page_count,categories,image_small,image_large,language,
    sale_country,list_price_amount,list_price_currency,
    buy_link,web_reader_link,embeddable,grade
) FROM 'books.csv'
  WITH (FORMAT csv, HEADER true, NULL 'NA');
```

31. still in the db, alter the user table to include a grade level for the user
```
# alter user table to add grade
ALTER TABLE users
ADD COLUMN grade TEXT;
ALTER TABLE

# update the current users
UPDATE users SET grade = '3' WHERE username = 'alice';
UPDATE users SET grade = '4' WHERE username = 'bob_smith';
UPDATE users SET grade = '5' WHERE username = 'hstimac';
```

32. update the recommendations section of app.py
```
@app.get("/recommendations/{user_id}")
def recommendations(user_id: int, limit: int = 5):
    conn = get_db_connection()
    cur  = conn.cursor()

    
    cur.execute("SELECT grade FROM users WHERE user_id = %s", (user_id,))
    row = cur.fetchone()
    user_grade = row[0] if row and row[0] else None

    # If user.grade is NULL, derive grade(s) from their rated books
    if not user_grade:
        cur.execute("""
            SELECT DISTINCT b.grade
            FROM   user_books ub
            JOIN   books b
              ON  (ub.isbn_10 IS NOT NULL AND ub.isbn_10 = b.isbn_10)
               OR (LOWER(ub.title)  = LOWER(b.title)
               AND LOWER(ub.author) = LOWER(b.authors))
            WHERE  ub.user_id = %s
              AND  ub.rating IS NOT NULL
        """, (user_id,))
        grades = [g[0] for g in cur.fetchall() if g[0] is not None]
    else:
        grades = [user_grade]

    
    if grades:
        grade_filter_sql = "b.grade IN %s"
        grade_filter_val = (tuple(grades),)
    else:
        # no grade info – use all grades
        grade_filter_sql = "TRUE"
        grade_filter_val = tuple()

    cur.execute(f"""
        WITH candidate AS (
            SELECT b.isbn_10, b.title, b.authors, b.grade,
                   COALESCE(AVG(ub2.rating),0) AS avg_rating,
                   COUNT(ub2.rating)           AS num_ratings
            FROM books b
            LEFT JOIN user_books ub2 ON b.isbn_10 = ub2.isbn_10
            WHERE {grade_filter_sql}
              AND NOT EXISTS (
                SELECT 1 FROM user_books ub
                WHERE  ub.user_id = %s
                  AND (
                       (ub.isbn_10 IS NOT NULL AND ub.isbn_10 = b.isbn_10)
                    OR (LOWER(ub.title)  = LOWER(b.title)
                    AND LOWER(ub.author) = LOWER(b.authors))
                  )
              )
            GROUP BY b.isbn_10, b.title, b.authors, b.grade
        )
        SELECT * FROM candidate
        ORDER BY avg_rating DESC, num_ratings DESC
        LIMIT %s;
    """, grade_filter_val + (user_id, limit))

    rows = cur.fetchall()

    
    if not rows:
        cur.execute("""
            SELECT title, authors, grade
            FROM   books
            ORDER  BY RANDOM()
            LIMIT  %s;
        """, (limit,))
        rows = cur.fetchall()

    cols = [d[0] for d in cur.description]
    cur.close(); conn.close()
    return [dict(zip(cols, r)) for r in rows]
```
*REMINDER*: Rerun uvicorn in terminal after changing app.py
```
uvicorn app:app --host=0.0.0.0 --port=8080 --reload
```

# Connect Vertext AI and BigQuery 
1. enable services in the shell terminal
```
gcloud services enable \
    aiplatform.googleapis.com \
    bigquery.googleapis.com \
    eventarc.googleapis.com \
    pubsub.googleapis.com \
    cloudfunctions.googleapis.com
```

2. set up BigQuery
```
# create a dataset
bq mk book_recs

# create embeddings table
bq query --use_legacy_sql=false """
CREATE OR REPLACE TABLE book_recs.book_embeddings (
  isbn_10 STRING,
  title STRING,
  embedding ARRAY<FLOAT64>
)
"""
```

3. create a pub/sub topic for new books and for rating books
```
gcloud pubsub topics create new-book
gcloud pubsub topics create rate-book
```

4. update POST /books in app.py with pub/sub
```
from google.cloud import pubsub_v1
import json

publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path("book-recommendations-456120", "new-book")

def publish_new_book(book_data: dict):
    payload = json.dumps({
        "isbn_10": book_data["isbn_10"],
        "description": book_data["description"]
    }).encode("utf-8")

    future = publisher.publish(topic_path, payload)
    print(f" Published to Pub/Sub: {future.result()}")
```
5. make a new directory and folder
```
mkdir ~/rate-book-function
cd ~/rate-book-function
```

7. create main.py in the rate-book-function
```
import base64
import json
import os
from google.cloud import storage
import psycopg2

DB_USER = "postgres"
DB_PASS = "stimac-cis655-final"
DB_NAME = "book_recommendations_db"
DB_HOST = "/cloudsql/book-recommendations-456120:us-central1:book-recs-db"

def get_conn():
    return psycopg2.connect(
        dbname=DB_NAME,
        user=DB_USER,
        password=DB_PASS,
        host=DB_HOST
    )

def entry_point(event, context):
    data = json.loads(base64.b64decode(event["data"]).decode("utf-8"))

    isbn_10 = data.get("isbn_10")
    title   = data["title"]
    author  = data["author"]
    user_id = data["user_id"]
    rating  = data.get("rating")
    status  = data.get("status")
    progress = data.get("progress")
    notes   = data.get("notes")

    conn = get_conn()
    cur = conn.cursor()

    try:
        # 1. Insert book if it doesn't exist
        cur.execute("""
            SELECT 1 FROM books WHERE isbn_10 = %s
        """, (isbn_10,))
        if not cur.fetchone():
            cur.execute("""
                INSERT INTO books (title, authors, isbn_10)
                VALUES (%s, %s, %s)
            """, (title, author, isbn_10))
            print(f" Added new book: {title}")

        # 2. Insert into user_books
        cur.execute("""
            INSERT INTO user_books (user_id, title, author, isbn_10, rating, status, progress, notes)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
        """, (user_id, title, author, isbn_10, rating, status, progress, notes))
        print(f" Added rating: {rating} for user {user_id}")

        conn.commit()

    except Exception as e:
        conn.rollback()
        print(f" Error: {e}")
    finally:
        cur.close()
        conn.close()
```

8. create a requirements.txt in the rate-book-function
```
psycopg2-binary
google-cloud-storage
```

9. deploy the function
```
gcloud functions deploy rate-book-writer \
  --runtime python311 \
  --entry-point entry_point \
  --trigger-topic rate-book \
  --region us-central1 \
  --memory 512MB \
  --timeout 60s \
  --source . \
  --no-gen2 \
  --set-env-vars GCP_PROJECT=book-recommendations-456120
```

10. test the function
```
gcloud pubsub topics publish rate-book \
  --message='{
    "user_id": 1,
    "title": "Charlotte’s Web",
    "author": "E. B. White",
    "isbn_10": "0064400557",
    "rating": 5,
    "status": "finished",
    "progress": 100,
    "notes": "So sweet!"
  }'
```

11. update app.py to connect to pub/sub, add to bottom of code
```
from google.cloud import pubsub_v1
import json

def publish_rating_event(entry: dict):
    publisher = pubsub_v1.PublisherClient()
    topic_path = publisher.topic_path("book-recommendations-456120", "rate-book")
    payload = json.dumps(entry).encode("utf-8")
    publisher.publish(topic_path, payload)
```

12. replace add_user_books in app.py
```
@app.post("/user_books")
def add_user_book(entry: UserBook):
    # Publish the rating event to Pub/Sub first
    try:
        publish_rating_event(entry.dict())
        return {"message": "Rating submitted to Pub/Sub successfully."}
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Failed to publish rating event: {str(e)}")
```

13. add pub.sub to requirements.txt
```
google-cloud-pubsub
```

14. install pub/sub if not installed in shell
```
pip install google-cloud-pubsub
```


15. create embeddings for Vertex AI
```
# enable Vertext AI in the shell termnial
gcloud services enable aiplatform.googleapis.com

# install the SDK in the terminal
pip install google-cloud-aiplatform
```

16. add to requirements.txt
```
google-cloud-aiplatform
```

17. open editor and create file generate_embeddings.py inside book-api
```
import vertexai
from vertexai.language_models import TextEmbeddingModel
from google.cloud import bigquery

# Constants
PROJECT_ID = "book-recommendations-456120"
REGION = "us-central1"
MODEL_ID = "text-embedding-005" 

vertexai.init(project=PROJECT_ID, location=REGION)
model = TextEmbeddingModel.from_pretrained(MODEL_ID)
bq_client = bigquery.Client()

# Example book description
title = "Matilda"
isbn_10 = "042528767X"
text = "Matilda is a story about a brilliant young girl with telekinesis and awful parents."

# Generate embedding
embedding = model.get_embeddings([text])[0].values

# Insert into BigQuery
row = {"isbn_10": isbn_10, "title": title, "embedding": embedding}
errors = bq_client.insert_rows_json(f"{PROJECT_ID}.book_recs.book_embeddings", [row])

if errors:
    print(" BigQuery insert failed:", errors)
else:
    print(f" Inserted embedding for: {title}")
```

19. in console, click Navigation >> Vertex AI >> Dashboard >> enable all recommended APIs

20. run in shell, make sure you're in correct directory
```
cd ~/book-api
python generate_embeddings.py
```

21. update generate_embeddings.py to loop through all books and generate embeddings
```
import vertexai
from vertexai.language_models import TextEmbeddingModel
from google.cloud import bigquery
import psycopg2

# Constants
PROJECT_ID = "book-recommendations-456120"
REGION = "us-central1"
MODEL_ID = "text-embedding-005" 

vertexai.init(project=PROJECT_ID, location=REGION)
model = TextEmbeddingModel.from_pretrained(MODEL_ID)
bq_client = bigquery.Client()

# DB connection settings
conn = psycopg2.connect(
    dbname="book_recommendations_db",
    user="postgres",
    password="stimac-cis655-final",
    host="127.0.0.1",
    port="5432"
)
cur = conn.cursor()

# Query books with non-empty descriptions
cur.execute("""
    SELECT isbn_10, title, description
    FROM books
    WHERE description IS NOT NULL AND LENGTH(description) > 50
""")

books = cur.fetchall()

print(f" Found {len(books)} books to embed")

for isbn_10, title, description in books:
    try:
        embedding = model.get_embeddings([description])[0].values

        row = {
            "isbn_10": isbn_10,
            "title": title,
            "embedding": embedding
        }

        errors = bq_client.insert_rows_json("book-recommendations-456120.book_recs.book_embeddings", [row])

        if errors:
            print(f" Failed to insert {title}: {errors}")
        else:
            print(f" Inserted: {title}")

    except Exception as e:
        print(f" Error for {title}: {e}")

cur.close()
conn.close()
```

22. rereun it in shell
```
python generate_embeddings.py
```

23. click Navigation >> BigQuery >> Studio >> book-recommendations-456120 (project name)

24. press the blue plus sign or the SQL query button to open a blank query page

25. enter the below code and press RUN
```
DECLARE target_isbn STRING DEFAULT "042528767X";  # must be an isbn_10 from embeddings

WITH
target AS (
  SELECT embedding
  FROM `book-recommendations-456120.book_recs.book_embeddings`
  WHERE isbn_10 = target_isbn
),

scored AS (
  SELECT
    b2.isbn_10,
    b2.title,
    b2.embedding,

    (
      SELECT SUM(e1 * e2)
      FROM UNNEST(b2.embedding) AS e1 WITH OFFSET i
      JOIN UNNEST(target.embedding) AS e2 WITH OFFSET j
      ON i = j
    ) /
    (
      SQRT(
        (SELECT SUM(POW(val, 2)) FROM UNNEST(b2.embedding) AS val)
      ) *
      SQRT(
        (SELECT SUM(POW(val2, 2)) FROM UNNEST(target.embedding) AS val2)
      )
    ) AS similarity

  FROM `book-recommendations-456120.book_recs.book_embeddings` b2, target
  WHERE b2.isbn_10 != target_isbn
)

SELECT *
FROM scored
ORDER BY similarity DESC
LIMIT 10;
```

26. turn this into an api endpoint by adding to app.py, place at bottom of current code
```
from fastapi import Path
from google.cloud import bigquery


@app.get("/recommendations/by-book")
def recommend_by_book(
    isbn_10: str = Query(None, description="ISBN-10 of the book"),
    title: str = Query(None, description="Title of the book"),
    authors: str = Query(None, description="Author(s) of the book"),
    grade: str = Query(None, description="Grade level to filter"),
    limit: int = Query(10, description="Max number of recommendations")
):
    try:
        client = bigquery.Client()

        if isbn_10:
            match_condition = f"isbn_10 = \"{isbn_10}\""
            exclude_condition = f"b2.isbn_10 != \"{isbn_10}\""
        elif title and authors:
            match_condition = f"LOWER(title) = LOWER(\"{title}\") AND LOWER(authors) = LOWER(\"{authors}\")"
            exclude_condition = "TRUE"
        else:
            raise HTTPException(status_code=400, detail="Must provide either ISBN-10 or both title and authors.")

        filter_grade = f"AND b2.grade = \"{grade}\"" if grade else ""

        query = f"""
        WITH
        target AS (
          SELECT embedding
          FROM `book-recommendations-456120.book_recs.book_embeddings`
          WHERE {match_condition}
        ),

        scored AS (
          SELECT
            b2.isbn_10,
            b2.title,
            b2.authors,
            b2.grade,
            (
              SELECT SUM(e1 * e2)
              FROM UNNEST(b2.embedding) AS e1 WITH OFFSET i
              JOIN UNNEST(target.embedding) AS e2 WITH OFFSET j
              ON i = j
            ) /
            (
              SQRT((SELECT SUM(POW(val, 2)) FROM UNNEST(b2.embedding) AS val)) *
              SQRT((SELECT SUM(POW(val2, 2)) FROM UNNEST(target.embedding) AS val2))
            ) AS similarity
          FROM `book-recommendations-456120.book_recs.book_embeddings` b2, target
          WHERE {exclude_condition} {filter_grade}
        )

        SELECT *
        FROM scored
        ORDER BY similarity DESC
        LIMIT {limit};
        """

        job = client.query(query)
        results = job.result()
        return [dict(row) for row in results]

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

27. run in shell terminal
```
# make sure in right directory and in virtual environment
cd ~/book-api
source venv/bin/activate

# install if not already installed
pip install google-cloud-bigquery

# reload the app
uvicorn app:app --host=0.0.0.0 --port=8080 --reload

```

28. change recommendations/user_id line in app.py to avoid issue
```
@app.get("/recommendations/user/{user_id}")
```

29. update BigQuery book_embeddings table to fix an error: Navigation >> BigQuery>> Studio >> select project name >> book_recs >> book_embeddings >> click Edit Schema button >> make sure you have title, author, grade, isbn_10 all as string nullable and embeddings as float repeated

30. export/extract more variables to enhance recommendations
```
# export 
gcloud sql export csv book-recs-db \
  gs://k8-books-bucket/book.csv \
  --database=book_recommendations_db \
  --query="SELECT isbn_10, title, authors, grade, description FROM books WHERE isbn_10 IS NOT NULL AND title IS NOT NULL" \
  --offload

# if denied, grant permission, find email
gcloud sql instances describe book-recs-db \
  --format="value(serviceAccountEmailAddress)"

# then grant permission 
gsutil iam ch \
  serviceAccount:p968113828557-ubbsf9@gcp-sa-cloud-sql.iam.gserviceaccount.com:roles/storage.objectAdmin \
  gs://k8-books-bucket

# then rerun export command
```

31.  import/update BigQuery
```
bq load \
  --source_format=CSV \
  --skip_leading_rows=1 \
  book_recs.book_embeddings \
  gs://k8-books-bucket/book.csv \
  isbn_10:STRING,title:STRING,authors:STRING,grade:STRING,description:STRING
```
32.  update generate_embeddings.py to fix error of missing variables
```
import vertexai
from vertexai.language_models import TextEmbeddingModel
from google.cloud import bigquery
import psycopg2

# Constants
PROJECT_ID = "book-recommendations-456120"
REGION = "us-central1"
MODEL_ID = "text-embedding-005" 

# Initialize Vertex AI and BigQuery client
vertexai.init(project=PROJECT_ID, location=REGION)
model = TextEmbeddingModel.from_pretrained(MODEL_ID)
bq_client = bigquery.Client()

# DB connection
conn = psycopg2.connect(
    dbname="book_recommendations_db",
    user="postgres",
    password="stimac-cis655-final",
    host="127.0.0.1",
    port="5432"
)
cur = conn.cursor()

# Query books with sufficient description
cur.execute("""
    SELECT isbn_10, title, authors, grade, description
    FROM books
    WHERE description IS NOT NULL AND LENGTH(description) > 10
""")

books = cur.fetchall()

print(f"Found {len(books)} books to embed")

for isbn_10, title, authors, grade, description in books:
    try:
        embedding = model.get_embeddings([description])[0].values

        row = {
            "isbn_10": isbn_10,
            "title": title,
            "authors": authors,
            "grade": grade,
            "description": description,
            "embedding": embedding
        }

        errors = bq_client.insert_rows_json(
            "book-recommendations-456120.book_recs.book_embeddings",
            [row]
        )

        if errors:
            print(f"Failed to insert {title}: {errors}")
        else:
            print(f"Inserted: {title}")

    except Exception as e:
        print(f"Error processing {title}: {e}")

cur.close()
conn.close()
```

33.  rerun in shell
```
python generate_embeddings.py
```

34.  updated BigQuery SQL query (named mine bigquery sql template)
```
DECLARE target_isbn STRING DEFAULT "0375866116";
DECLARE target_title STRING DEFAULT NULL;
DECLARE target_authors STRING DEFAULT NULL;
DECLARE target_grade STRING DEFAULT NULL;
DECLARE target_keyword STRING DEFAULT NULL;

WITH target AS (
  SELECT embedding
  FROM `book-recommendations-456120.book_recs.book_embeddings`
  WHERE (
    (target_isbn IS NOT NULL AND isbn_10 = target_isbn) OR
    (target_title IS NOT NULL AND LOWER(title) = LOWER(target_title)) OR
    (target_authors IS NOT NULL AND LOWER(authors) = LOWER(target_authors)) OR
    (target_grade IS NOT NULL AND grade = target_grade) OR
    (target_keyword IS NOT NULL AND (
      LOWER(title) LIKE CONCAT('%', LOWER(target_keyword), '%') OR
      LOWER(authors) LIKE CONCAT('%', LOWER(target_keyword), '%') OR
      LOWER(description) LIKE CONCAT('%', LOWER(target_keyword), '%')
    ))
  )
  LIMIT 1
),

scored AS (
  SELECT
    b2.isbn_10,
    b2.title,
    b2.authors,
    b2.grade,
    (
      SELECT SUM(e1 * e2)
      FROM UNNEST(b2.embedding) AS e1 WITH OFFSET i
      JOIN UNNEST(target.embedding) AS e2 WITH OFFSET j
      ON i = j
    ) /
    (
      SQRT((SELECT SUM(POW(val, 2)) FROM UNNEST(b2.embedding) AS val)) *
      SQRT((SELECT SUM(POW(val2, 2)) FROM UNNEST(target.embedding) AS val2))
    ) AS similarity
  FROM `book-recommendations-456120.book_recs.book_embeddings` b2, target
  WHERE b2.embedding IS NOT NULL
    AND (
      target_isbn IS NULL OR b2.isbn_10 != target_isbn
    )
)

SELECT *
FROM scored
ORDER BY similarity DESC
LIMIT 10;
```

35.  update app.py and replace the recommendations by book bigquery endpoint
```
from fastapi import Query
from google.cloud import bigquery
from fastapi import FastAPI, HTTPException
from typing import Optional

@app.get("/recommendations/by-metadata")
def recommend_by_metadata(
    isbn_10: Optional[str] = Query(None),
    title: Optional[str] = Query(None),
    authors: Optional[str] = Query(None),
    grade: Optional[str] = Query(None),
    keyword: Optional[str] = Query(None),
    limit: int = Query(10, description="Number of recommendations to return")
):
    try:
        client = bigquery.Client()

        filters = []
        if isbn_10:
            filters.append(f"LOWER(isbn_10) = LOWER(@isbn_10)")
        if title:
            filters.append(f"LOWER(REGEXP_REPLACE(title, r'[^a-zA-Z0-9]', '')) = LOWER(REGEXP_REPLACE(@title, r'[^a-zA-Z0-9]', ''))")
        if authors:
            filters.append(f"LOWER(REGEXP_REPLACE(authors, r'[^a-zA-Z0-9]', '')) = LOWER(REGEXP_REPLACE(@authors, r'[^a-zA-Z0-9]', ''))")
        if grade:
            filters.append(f"grade = @grade")
        if keyword:
            filters.append(
                """(
                    LOWER(title) LIKE CONCAT('%', LOWER(@keyword), '%') OR
                    LOWER(authors) LIKE CONCAT('%', LOWER(@keyword), '%') OR
                    LOWER(description) LIKE CONCAT('%', LOWER(@keyword), '%')
                )"""
            )

        if not filters:
            raise HTTPException(status_code=400, detail="At least one filter must be provided.")

        where_clause = " AND ".join(filters)

        query = f"""
        WITH target AS (
            SELECT embedding
            FROM `book-recommendations-456120.book_recs.book_embeddings`
            WHERE {where_clause}
            LIMIT 1
        ),
        scored AS (
            SELECT
                b2.isbn_10,
                b2.title,
                b2.authors,
                b2.grade,
                (
                    SELECT SUM(e1 * e2)
                    FROM UNNEST(b2.embedding) AS e1 WITH OFFSET i
                    JOIN UNNEST(target.embedding) AS e2 WITH OFFSET j
                    ON i = j
                ) /
                (
                    SQRT((SELECT SUM(POW(val, 2)) FROM UNNEST(b2.embedding) AS val)) *
                    SQRT((SELECT SUM(POW(val2, 2)) FROM UNNEST(target.embedding) AS val2))
                ) AS similarity
            FROM `book-recommendations-456120.book_recs.book_embeddings` b2, target
            WHERE b2.embedding IS NOT NULL
        )
        SELECT *
        FROM scored
        ORDER BY similarity DESC
        LIMIT @limit
        """

        job_config = bigquery.QueryJobConfig(
            query_parameters=[
                bigquery.ScalarQueryParameter("isbn_10", "STRING", isbn_10),
                bigquery.ScalarQueryParameter("title", "STRING", title),
                bigquery.ScalarQueryParameter("authors", "STRING", authors),
                bigquery.ScalarQueryParameter("grade", "STRING", grade),
                bigquery.ScalarQueryParameter("keyword", "STRING", keyword),
                bigquery.ScalarQueryParameter("limit", "INT64", limit),
            ]
        )

        job = client.query(query, job_config=job_config)
        results = job.result()
        return [dict(row) for row in results]

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

36. reload the app
```
uvicorn app:app --host=0.0.0.0 --port=8080 --reload
```

37. create a json file for the bigquery table named book_embeddings_schema.json
```
[
    { "name": "isbn_10", "type": "STRING", "mode": "NULLABLE" },
    { "name": "title", "type": "STRING", "mode": "NULLABLE" },
    { "name": "authors", "type": "STRING", "mode": "NULLABLE" },
    { "name": "grade", "type": "STRING", "mode": "NULLABLE" },
    { "name": "embedding", "type": "FLOAT64", "mode": "REPEATED" },
    { "name": "description", "type": "STRING", "mode": "NULLABLE" }
  ]
```
38. update it in shell
```
bq update \
  --table \
  book-recommendations-456120:book_recs.book_embeddings \
  book_embeddings_schema.json
```

# Create the Vertex AI chatbox
1. install in shell if not already installed
```
pip install google-cloud-aiplatform
```

2. in Editor, create a chatbot.py file in book-api
```
# chatbot.py
import vertexai
from vertexai.language_models import ChatModel, InputOutputTextPair

vertexai.init(
    project="book-recommendations-456120", 
    location="us-central1"
)

chat_model = ChatModel.from_pretrained("chat-bison@002")

chat = chat_model.start_chat(
    context="You are a friendly book assistant. Recommend books based on what the user says. Use genres, topics, and reading level if given.",
)

def ask_chatbot(prompt: str) -> str:
    response = chat.send_message(prompt)
    return response.text
```

3. add to app.py
```
## AI Chatbot

from chatbot import ask_chatbot
from fastapi import Body

@app.post("/chat")
def chat_with_user(message: str = Body(..., embed=True)):
    try:
        response = ask_chatbot(message)
        return {"response": response}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

4. reload the app
```
uvicorn app:app --host=0.0.0.0 --port=8080 --reload
```

5. PAUSE: ran into issue with vertex ai, awaiting feeback

#####

# Add feedback/reaction endpoint for users interacting with recommendations
1. switch to db
```
psql -h 127.0.0.1 -U postgres -d book_recommendations_db
```

2. create table for feedback
```
CREATE TABLE recommendation_feedback (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(user_id),
    isbn_10 TEXT,
    feedback TEXT CHECK (feedback IN ('like', 'dislike', 'interested')),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

3. add feedback endpoint to app.py
```
## Feeback endpoint
from datetime import datetime
from google.cloud import bigquery

class Feedback(BaseModel):
    user_id: int
    isbn_10: str
    feedback: str  # 'like', 'dislike', 'interested'

@app.post("/feedback")
def submit_feedback(entry: Feedback):
    if entry.feedback not in ['like', 'dislike', 'interested']:
        raise HTTPException(status_code=400, detail="Invalid feedback type.")

    # Store feedback in PostgreSQL
    conn = get_db_connection()
    cur = conn.cursor()
    try:
        cur.execute("""
            INSERT INTO recommendation_feedback (user_id, isbn_10, feedback)
            VALUES (%s, %s, %s)
        """, (entry.user_id, entry.isbn_10, entry.feedback))
        conn.commit()
    except Exception as e:
        conn.rollback()
        raise HTTPException(status_code=500, detail=f"PostgreSQL error: {str(e)}")
    finally:
        cur.close()
        conn.close()

    # Also update BigQuery thumbs_up/thumbs_down
    try:
        if entry.feedback in ['like', 'dislike']:
            bq_client = bigquery.Client()
            field = "thumbs_up" if entry.feedback == "like" else "thumbs_down"

            query = f"""
                UPDATE `book-recommendations-456120.book_recs.book_embeddings`
                SET {field} = IFNULL({field}, 0) + 1
                WHERE isbn_10 = @isbn_10
            """

            job = bq_client.query(query, job_config=bigquery.QueryJobConfig(
                query_parameters=[
                    bigquery.ScalarQueryParameter("isbn_10", "STRING", entry.isbn_10)
                ]
            ))
            job.result()

    except Exception as e:
        raise HTTPException(status_code=500, detail=f"BigQuery error: {str(e)}")

    return {"message": "Feedback submitted successfully."}
```

4. update book_embeddings_schema.json
```
[
    {"name": "isbn_10", "type": "STRING"},
    {"name": "title", "type": "STRING"},
    {"name": "authors", "type": "STRING"},
    {"name": "description", "type": "STRING"},
    {"name": "grade", "type": "STRING"},
    {"name": "embedding", "type": "FLOAT64", "mode": "REPEATED"},
    {"name": "thumbs_up", "type": "INTEGER"},
    {"name": "thumbs_down", "type": "INTEGER"}
  ]
```

5. update bigquery table in shell
```
bq update \
  --table book-recommendations-456120:book_recs.book_embeddings \
  ./book_embeddings_schema.json
```

6. update the sql query (BigQuery >> Studio >> project name >> queries (select the one you've been using)
```
DECLARE target_isbn STRING DEFAULT "0375866116";
DECLARE target_title STRING DEFAULT NULL;
DECLARE target_authors STRING DEFAULT NULL;
DECLARE target_grade STRING DEFAULT NULL;
DECLARE target_keyword STRING DEFAULT NULL;

WITH target AS (
  SELECT embedding
  FROM `book-recommendations-456120.book_recs.book_embeddings`
  WHERE (
    (target_isbn IS NOT NULL AND isbn_10 = target_isbn) OR
    (target_title IS NOT NULL AND LOWER(title) = LOWER(target_title)) OR
    (target_authors IS NOT NULL AND LOWER(authors) = LOWER(target_authors)) OR
    (target_grade IS NOT NULL AND grade = target_grade) OR
    (target_keyword IS NOT NULL AND (
      LOWER(title) LIKE CONCAT('%', LOWER(target_keyword), '%') OR
      LOWER(authors) LIKE CONCAT('%', LOWER(target_keyword), '%') OR
      LOWER(description) LIKE CONCAT('%', LOWER(target_keyword), '%')
    ))
  )
  LIMIT 1
),

scored AS (
  SELECT
    b2.isbn_10,
    b2.title,
    b2.authors,
    b2.grade,
    (
      SELECT SUM(e1 * e2)
      FROM UNNEST(b2.embedding) AS e1 WITH OFFSET i
      JOIN UNNEST(target.embedding) AS e2 WITH OFFSET j
      ON i = j
    ) /
    (
      SQRT((SELECT SUM(POW(val, 2)) FROM UNNEST(b2.embedding) AS val)) *
      SQRT((SELECT SUM(POW(val2, 2)) FROM UNNEST(target.embedding) AS val2))
    ) AS similarity
  FROM `book-recommendations-456120.book_recs.book_embeddings` b2, target
  WHERE b2.embedding IS NOT NULL
    AND (
      target_isbn IS NULL OR b2.isbn_10 != target_isbn
    )
)

SELECT *
FROM scored
ORDER BY similarity DESC
LIMIT 10;
```

7. in app.py update @app.get("/recommendations/by-metadata") to include thumbsUp and thumbs_down
```
 # everything above this stays the same       
            SELECT
                b2.isbn_10,
                b2.title,
                b2.authors,
                b2.thumbs_up,
                b2.thumbs_down,
                b2.grade,
# everything below this stays the same
```

8. reload the app in shell
```
uvicorn app:app --host=0.0.0.0 --port=8080 --reload
```

# Build the Front End

1. make a static folder inside book-api
```
mkdir static
```

2. in Editor, select static folder and create file index.html

3. inside index.html add code
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Book Recommendation App</title>
    <style>
        body { font-family: Arial, sans-serif; background: #f9f9f9; margin: 0; padding: 20px; }
        h1, h2 { color: #333; }
        .section { margin-bottom: 40px; }
        input, button, select { padding: 8px; margin: 5px; font-size: 16px; }
        .book { border: 1px solid #ccc; border-radius: 8px; background: #fff; padding: 10px; margin: 10px 0; display: flex; align-items: flex-start; }
        .book img { height: 100px; margin-right: 15px; }
        .book a { display: inline-block; margin-top: 10px; color: blue; text-decoration: underline; }
    </style>
</head>
<body>
    <h1>Book Recommender</h1>

    <!-- Login Section -->
    <div class="section">
        <h2>Login</h2>
        <input type="text" id="username" placeholder="Username">
        <input type="email" id="email" placeholder="Email">
        <button onclick="loginUser()">Login / Register</button>
    </div>

    <!-- Search Section -->
    <div class="section">
        <h2>Search Books</h2>
        <input type="text" id="searchQuery" placeholder="Title, Author, or Keyword">
        <button onclick="searchBooks()">Search</button>
        <div id="searchResults"></div>
    </div>

    <!-- Recommendation Section -->
    <div class="section">
        <h2>Get Recommendations</h2>
        <button onclick="getRecommendations()">Get Recommendations</button>
        <div id="recommendations"></div>
    </div>

    <script>
        let currentUserId = null;

        function loginUser() {
            const username = document.getElementById('username').value;
            const email = document.getElementById('email').value;
            fetch('/users', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ username, email })
            })
            .then(res => res.json())
            .then(data => {
                currentUserId = data[0];
                alert('Logged in as ' + username);
            });
        }

        function searchBooks() {
            const query = document.getElementById('searchQuery').value;
            fetch(`/books?title=${encodeURIComponent(query)}`)
                .then(res => res.json())
                .then(books => {
                    const resultsDiv = document.getElementById('searchResults');
                    resultsDiv.innerHTML = '';
                    books.forEach(book => {
                        const div = document.createElement('div');
                        div.className = 'book';
                        div.innerHTML = `
                            <img src="${book.image_small || ''}" alt="cover">
                            <div>
                                <strong>${book.title}</strong><br>
                                <em>${book.authors}</em><br>
                                <button onclick="rateBook('${book.isbn_10}', '${book.title}', '${book.authors}', 5)">👍</button>
                                <button onclick="rateBook('${book.isbn_10}', '${book.title}', '${book.authors}', 1)">👎</button><br>
                                ${book.buy_link ? `<a href="${book.buy_link}" target="_blank">Buy</a>` : ''}
                            </div>
                        `;
                        resultsDiv.appendChild(div);
                    });
                });
        }

        function rateBook(isbn_10, title, author, rating) {
            if (!currentUserId) return alert('Please log in first.');
            fetch('/user_books', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({
                    user_id: currentUserId,
                    isbn_10, title, author,
                    rating: rating,
                    status: 'completed'
                })
            })
            .then(() => alert('Rating submitted!'));
        }

        function getRecommendations() {
            if (!currentUserId) return alert('Please log in first.');
            fetch(`/recommendations/${currentUserId}`)
                .then(res => res.json())
                .then(books => {
                    const resultsDiv = document.getElementById('recommendations');
                    resultsDiv.innerHTML = '';
                    books.forEach(book => {
                        const div = document.createElement('div');
                        div.className = 'book';
                        div.innerHTML = `
                            <img src="" alt="cover">
                            <div>
                                <strong>${book.title}</strong><br>
                                <em>${book.authors}</em><br>
                                ${book.buy_link ? `<a href="${book.buy_link}" target="_blank">Buy</a>` : ''}
                            </div>
                        `;
                        resultsDiv.appendChild(div);
                    });
                });
        }
    </script>
</body>
</html>
```

4. open app.py and add code chunk near top under app = FastAPI()
```
from fastapi.staticfiles import StaticFiles
from fastapi.responses import FileResponse

# Serve static files
app.mount("/static", StaticFiles(directory="static"), name="static")

# Route for the homepage
@app.get("/")
def read_index():
    return FileResponse("static/index.html")
```

5. rerun app
```
uvicorn app:app --host=0.0.0.0 --port=8080 --reload
```
 
# Cloud Run

1. in book-api in shell, copy all current dependencies to a new file (it will create the file)
```
pip freeze > requirements.txt
```

2. create a Docker file
```
# enter into shell
touch Dockerfile
```

3. open Editor, select Dockerfile, add code
```
# Use official Python image
FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Copy requirements and install dependencies
COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

# Copy rest of the app (code, templates, etc.)
COPY . .

# Expose port
EXPOSE 8080

# Run the FastAPI app with uvicorn
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8080"]
```

4. use shell terminal to deploy to Cloud Run
```
gcloud builds submit --tag gcr.io/book-recommendations-456120/book-api
gcloud run deploy book-api \
  --image gcr.io/book-recommendations-456120/book-api \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated
```

# Build user authenticaion and better security

1. add to requirements.txt
```
python-jose
passlib[bcrypt]
```

2. run in shell
```
pip install -r requirements.txt
```

3. in Editor, create auth.py in book-api for user authentication

4. put the code in auth.py to handle registration, login, and utilities for users
```
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from pydantic import BaseModel
from passlib.context import CryptContext
from jose import JWTError, jwt
from datetime import datetime, timedelta
from typing import Optional
from app import get_db_connection

router = APIRouter()

# Secret key and algorithm
SECRET_KEY = "your-secret-key"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

class User(BaseModel):
    username: str
    email: str
    password: str

class Token(BaseModel):
    access_token: str
    token_type: str

def get_password_hash(password):
    return pwd_context.hash(password)

def verify_password(plain_password, hashed_password):
    return pwd_context.verify(plain_password, hashed_password)

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None):
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

@router.post("/register")
def register(user: User):
    conn = get_db_connection()
    cur = conn.cursor()
    try:
        cur.execute("SELECT * FROM users WHERE username = %s OR email = %s", (user.username, user.email))
        if cur.fetchone():
            raise HTTPException(status_code=400, detail="Username or email already exists")

        hashed_password = get_password_hash(user.password)
        cur.execute(
            "INSERT INTO users (username, email, hashed_password) VALUES (%s, %s, %s)",
            (user.username, user.email, hashed_password)
        )
        conn.commit()
        return {"message": "User registered successfully"}
    except Exception as e:
        conn.rollback()
        raise HTTPException(status_code=500, detail=str(e))
    finally:
        cur.close()
        conn.close()

@router.post("/login", response_model=Token)
def login(form_data: OAuth2PasswordRequestForm = Depends()):
    conn = get_db_connection()
    cur = conn.cursor()
    try:
        cur.execute("SELECT user_id, username, email, hashed_password FROM users WHERE username = %s", (form_data.username,))
        user = cur.fetchone()
        if not user or not verify_password(form_data.password, user[3]):
            raise HTTPException(status_code=400, detail="Invalid credentials")

        access_token = create_access_token(data={"sub": user[1]})
        return {"access_token": access_token, "token_type": "bearer"}
    finally:
        cur.close()
        conn.close()
```

5. update `user` table
```
psql "host=127.0.0.1 dbname=book_recommendations_db user=postgres password=stimac-cis655-final"

ALTER TABLE users
ADD COLUMN password TEXT,
ADD COLUMN hashed_password TEXT;
```

6. update app.py near the top below FastAPI
```
from fastapi import Depends
from auth import router as auth_router, verify_token
from auth import router as auth_router

app.include_router(auth_router)
```

```
# update these lines in app.py to add authentication

@app.get("/user_books/{user_id}", dependencies=[Depends(verify_token)])

@app.post("/user_books", dependencies=[Depends(verify_token)])

@app.get("/recommendations/for-user/{user_id}", dependencies=[Depends(verify_token)])

@app.post("/feedback", dependencies=[Depends(verify_token)])

@app.post("/books", response_model=Book, dependencies=[Depends(verify_token)])

@app.put("/books/{isbn}", response_model=Book, dependencies=[Depends(verify_token)])

@app.delete("/books/{isbn}", dependencies=[Depends(verify_token)])
```

7. update index.html to include better search options and feedback input
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Book Recommendation App</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background: #f9f9f9;
            margin: 0;
            padding: 20px;
        }
        h1, h2 {
            color: #333;
        }
        .section {
            margin-bottom: 40px;
        }
        input, button, select {
            padding: 8px;
            margin: 5px;
            font-size: 16px;
        }
        .book {
            border: 1px solid #ccc;
            border-radius: 8px;
            background: #fff;
            padding: 10px;
            margin: 10px 0;
            display: flex;
            align-items: flex-start;
        }
        .book img {
            height: 100px;
            margin-right: 15px;
        }
        .book a {
            display: inline-block;
            margin-top: 10px;
            color: blue;
            text-decoration: underline;
        }
    </style>
</head>
<body>
    <h1>Book Recommender</h1>

    <!-- Login Section -->
    <div class="section">
        <h2>Login</h2>
        <input type="text" id="username" placeholder="Username">
        <input type="email" id="email" placeholder="Email">
        <button onclick="loginUser()">Login / Register</button>
    </div>

    <!-- Search Section -->
    <div class="section">
        <h2>Search Books (Basic)</h2>
        <input type="text" id="searchQuery" placeholder="Title, Author, or Keyword">
        <button onclick="searchBooks()">Search</button>
        <div id="searchResults"></div>
    </div>

    <!-- Smart Recommendation Section -->
    <div class="section">
        <h2>Find Book Recommendations</h2>
        <input type="text" id="recTitle" placeholder="Title">
        <input type="text" id="recAuthors" placeholder="Author(s)">
        <input type="text" id="recIsbn" placeholder="ISBN-10">
        <input type="text" id="recKeyword" placeholder="Keyword">
        <select id="recGrade">
            <option value="">Select Grade</option>
            <option value="K">K</option>
            <option value="1">1</option>
            <option value="2">2</option>
            <option value="3">3</option>
            <option value="4">4</option>
            <option value="5">5</option>
            <option value="6">6</option>
            <option value="7">7</option>
            <option value="8">8</option>
        </select>
        <button onclick="getMetadataRecommendations()">Get Recommendations</button>
        <div id="recommendations"></div>
    </div>

    <script>
        let currentUserId = null;

        function loginUser() {
            const username = document.getElementById('username').value;
            const email = document.getElementById('email').value;
            fetch('/users', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ username, email })
            })
            .then(res => res.json())
            .then(data => {
                currentUserId = data[0];
                alert('Logged in as ' + username);
            });
        }

        function searchBooks() {
            const query = document.getElementById('searchQuery').value;
            fetch(`/books?title=${encodeURIComponent(query)}`)
                .then(res => res.json())
                .then(books => {
                    const resultsDiv = document.getElementById('searchResults');
                    resultsDiv.innerHTML = '';
                    books.forEach(book => {
                        const div = document.createElement('div');
                        div.className = 'book';
                        div.innerHTML = `
                            <img src="${book.image_small || ''}" alt="cover">
                            <div>
                                <strong>${book.title}</strong><br>
                                <em>${book.authors}</em><br>
                                <button onclick="rateBook('${book.isbn_10}', '${book.title}', '${book.authors}', 'like')">👍</button>
                                <button onclick="rateBook('${book.isbn_10}', '${book.title}', '${book.authors}', 'dislike')">👎</button><br>
                                ${book.buy_link ? `<a href="${book.buy_link}" target="_blank">Buy</a>` : ''}
                            </div>
                        `;
                        resultsDiv.appendChild(div);
                    });
                });
        }

        function rateBook(isbn_10, title, author, feedback) {
            if (!currentUserId) return alert('Please log in first.');
            fetch('/feedback', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ user_id: currentUserId, isbn_10, feedback })
            })
            .then(() => alert('Feedback submitted!'));
        }

        function getMetadataRecommendations() {
            const title = document.getElementById('recTitle').value;
            const authors = document.getElementById('recAuthors').value;
            const isbn_10 = document.getElementById('recIsbn').value;
            const keyword = document.getElementById('recKeyword').value;
            const grade = document.getElementById('recGrade').value;

            const params = new URLSearchParams();
            if (title) params.append('title', title);
            if (authors) params.append('authors', authors);
            if (isbn_10) params.append('isbn_10', isbn_10);
            if (keyword) params.append('keyword', keyword);
            if (grade) params.append('grade', grade);

            fetch(`/recommendations/by-metadata?${params.toString()}`)
                .then(res => res.json())
                .then(books => {
                    const resultsDiv = document.getElementById('recommendations');
                    resultsDiv.innerHTML = '';
                    books.forEach(book => {
                        const div = document.createElement('div');
                        div.className = 'book';
                        div.innerHTML = `
                            <img src="${book.image_small || ''}" alt="cover">
                            <div>
                                <strong>${book.title}</strong><br>
                                <em>${book.authors}</em><br>
                                <button onclick="rateBook('${book.isbn_10}', '${book.title}', '${book.authors}', 'like')">👍</button>
                                <button onclick="rateBook('${book.isbn_10}', '${book.title}', '${book.authors}', 'dislike')">👎</button><br>
                                ${book.buy_link ? `<a href="${book.buy_link}" target="_blank">Buy</a>` : ''}
                            </div>
                        `;
                        resultsDiv.appendChild(div);
                    });
                })
                .catch(error => {
                    alert("Something went wrong with the recommendation request.");
                    console.error(error);
                });
        }
    </script>
</body>
</html>
```

8. ran into connection error. resolve by:
```
# create db.py file in book-api
import psycopg2
import os

def get_db_connection():
    return psycopg2.connect(
        dbname="book_recommendations_db",
        user="postgres",
        password="stimac-cis655-final",
        host="127.0.0.1",
        port="5432"
    )

# in app.py replace get_db_connection() with
from db import get_db_connection

# in auth.py add
from fastapi.security import OAuth2PasswordBearer
from db import get_db_connection 

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="login")

def verify_token(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, "your-secret-key", algorithms=["HS256"])
        return payload
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")
```

9. fix another error
```
# Reinstall locally if needed
pip install python-multipart

# Freeze updated dependencies
pip freeze > requirements.txt
```

10. redeploy to cloudrun to check functionality
```
# save to Docker file
gcloud builds submit --tag gcr.io/book-recommendations-456120/book-api

# redeploy Cloud Run
gcloud run deploy book-api \
  --image gcr.io/book-recommendations-456120/book-api \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --port 8080

```

11. update index.html to fix search
```
function searchBooks() {
    const query = document.getElementById('searchQuery').value;

    const params = new URLSearchParams();
    if (query) {
        params.append('title', query);
        params.append('authors', query);
        params.append('description', query);
        params.append('grade', query);
    }

    fetch(`/books?${params.toString()}`)
        .then(res => res.json())
        .then(books => {
            const resultsDiv = document.getElementById('searchResults');
            resultsDiv.innerHTML = '';
            books.forEach(book => {
                const div = document.createElement('div');
                div.className = 'book';
                div.innerHTML = `
                    <img src="${book.image_small || ''}" alt="cover">
                    <div>
                        <strong>${book.title}</strong><br>
                        <em>${book.authors}</em><br>
                        <button onclick="rateBook('${book.isbn_10}', '${book.title}', '${book.authors}', 'like')">👍</button>
                        <button onclick="rateBook('${book.isbn_10}', '${book.title}', '${book.authors}', 'dislike')">👎</button><br>
                        ${book.buy_link ? `<a href="${book.buy_link}" target="_blank">Buy</a>` : ''}
                    </div>
                `;
                resultsDiv.appendChild(div);
            });
        })
        .catch(error => {
            console.error('Search failed:', error);
        });
}


```
12. current index.html below

```
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Book Recommendation App</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      background: #f9f9f9;
      margin: 0;
      padding: 20px;
    }
    h1, h2 {
      color: #333;
    }
    .section {
      margin-bottom: 40px;
    }
    input, button, select {
      padding: 8px;
      margin: 5px;
      font-size: 16px;
    }
    .book {
      border: 1px solid #ccc;
      border-radius: 8px;
      background: #fff;
      padding: 10px;
      margin: 10px 0;
      display: flex;
      align-items: flex-start;
    }
    .book img {
      height: 100px;
      margin-right: 15px;
    }
    .book a {
      display: inline-block;
      margin-top: 10px;
      color: blue;
      text-decoration: underline;
    }
  </style>
</head>
<body>
  <h1>Book Recommender</h1>

  <!-- Login Section -->
  <div class="section">
    <h2>Login</h2>
    <input type="text" id="username" placeholder="Username">
    <input type="email" id="email" placeholder="Email">
    <button onclick="loginUser()">Login / Register</button>
  </div>

  <!-- Search Section -->
  <div class="section">
    <h2>Search Books</h2>
    <input type="text" id="searchTitle" placeholder="Title">
    <input type="text" id="searchAuthors" placeholder="Author">
    <input type="text" id="searchKeyword" placeholder="Keyword (description)">
    <select id="searchGrade">
      <option value="">Grade</option>
      <option value="K">K</option>
      <option value="1">1</option>
      <option value="2">2</option>
      <option value="3">3</option>
      <option value="4">4</option>
      <option value="5">5</option>
      <option value="6">6</option>
      <option value="7">7</option>
      <option value="8">8</option>
    </select>
    <button onclick="searchBooks()">Search</button>
    <div id="searchResults"></div>
  </div>

  <!-- Metadata Recommendation Section -->
  <div class="section">
    <h2>Find Book Recommendations</h2>
    <input type="text" id="recTitle" placeholder="Title">
    <input type="text" id="recAuthors" placeholder="Author(s)">
    <input type="text" id="recIsbn" placeholder="ISBN-10">
    <input type="text" id="recKeyword" placeholder="Keyword">
    <select id="recGrade">
      <option value="">Select Grade</option>
      <option value="K">K</option>
      <option value="1">1</option>
      <option value="2">2</option>
      <option value="3">3</option>
      <option value="4">4</option>
      <option value="5">5</option>
      <option value="6">6</option>
      <option value="7">7</option>
      <option value="8">8</option>
    </select>
    <button onclick="getMetadataRecommendations()">Get Recommendations</button>
    <div id="recommendations"></div>
  </div>

  <script>
    let currentUserId = null;

    function loginUser() {
      const username = document.getElementById('username').value;
      const email = document.getElementById('email').value;

      fetch('/users', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ username, email })
      })
      .then(res => res.json())
      .then(data => {
        currentUserId = data.user_id;
        alert('Logged in as ' + data.username);
      })
      .catch(error => {
        console.error("Login failed:", error);
        alert("Login failed. Please try again.");
      });
    }

    function searchBooks() {
      const title = document.getElementById('searchTitle').value;
      const authors = document.getElementById('searchAuthors').value;
      const keyword = document.getElementById('searchKeyword').value;
      const grade = document.getElementById('searchGrade').value;

      const params = new URLSearchParams();
      if (title) params.append('title', title);
      if (authors) params.append('authors', authors);
      if (keyword) params.append('description', keyword);
      if (grade) params.append('grade', grade);

      fetch(`/books?${params.toString()}`)
        .then(res => {
          if (!res.ok) throw new Error("Search failed");
          return res.json();
        })
        .then(books => {
          const resultsDiv = document.getElementById('searchResults');
          resultsDiv.innerHTML = '';
          books.forEach(book => {
            const div = document.createElement('div');
            div.className = 'book';
            div.innerHTML = `
              <img src="${book.image_small || '/static/placeholder.png'}" alt="cover">
              <div>
                <strong>${book.title}</strong><br>
                <em>${book.authors}</em><br>
                <button onclick="rateBook('${book.isbn_10}', '${book.title}', '${book.authors}', 'like')">👍</button>
                <button onclick="rateBook('${book.isbn_10}', '${book.title}', '${book.authors}', 'dislike')">👎</button><br>
                ${book.buy_link ? `<a href="${book.buy_link}" target="_blank">Buy</a>` : ''}
              </div>
            `;
            resultsDiv.appendChild(div);
          });
        })
        .catch(error => {
          console.error("Search error:", error);
          alert("Search failed.");
        });
    }

    function getMetadataRecommendations() {
      const title = document.getElementById('recTitle').value;
      const authors = document.getElementById('recAuthors').value;
      const isbn_10 = document.getElementById('recIsbn').value;
      const keyword = document.getElementById('recKeyword').value;
      const grade = document.getElementById('recGrade').value;

      const params = new URLSearchParams();
      if (title) params.append('title', title);
      if (authors) params.append('authors', authors);
      if (isbn_10) params.append('isbn_10', isbn_10);
      if (keyword) params.append('keyword', keyword);
      if (grade) params.append('grade', grade);

      fetch(`/recommendations/by-metadata?${params.toString()}`)
        .then(res => res.json())
        .then(books => {
          const resultsDiv = document.getElementById('recommendations');
          resultsDiv.innerHTML = '';
          books.forEach(book => {
            const div = document.createElement('div');
            div.className = 'book';
            div.innerHTML = `
              <img src="${book.image_small || '/static/placeholder.png'}" alt="cover">
              <div>
                <strong>${book.title}</strong><br>
                <em>${book.authors}</em><br>
                <button onclick="rateBook('${book.isbn_10}', '${book.title}', '${book.authors}', 'like')">👍</button>
                <button onclick="rateBook('${book.isbn_10}', '${book.title}', '${book.authors}', 'dislike')">👎</button><br>
                ${book.buy_link ? `<a href="${book.buy_link}" target="_blank">Buy</a>` : ''}
              </div>
            `;
            resultsDiv.appendChild(div);
          });
        })
        .catch(error => {
          alert("Recommendation failed.");
          console.error(error);
        });
    }

    function rateBook(isbn_10, title, author, feedback) {
      if (!currentUserId) return alert('Please log in first.');
      fetch('/feedback', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ user_id: currentUserId, isbn_10, feedback })
      })
      .then(() => alert('Feedback submitted!'))
      .catch(error => {
        console.error('Feedback failed:', error);
      });
    }
  </script>
</body>
</html>

```

13. update app.py to allow user reviews
```
class Review(BaseModel):
    user_id: int
    isbn_10: Optional[str]
    title: str
    author: str
    rating: Optional[int]
    notes: Optional[str]
    status: Optional[str] = "completed"
    reviewed_at: Optional[str] = None  # If not provided, set it to now

@app.post("/review", dependencies=[Depends(get_current_user)])
def submit_review(entry: Review):
    try:
        # If no timestamp is given, add it
        if not entry.reviewed_at:
            entry.reviewed_at = datetime.utcnow().isoformat()

        publisher = pubsub_v1.PublisherClient()
        topic_path = publisher.topic_path("book-recommendations-456120", "user-reviews")

        payload = json.dumps(entry.dict()).encode("utf-8")
        publisher.publish(topic_path, payload)

        return {"message": "Review submitted successfully."}

    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Error submitting review: {str(e)}")

```

14. fix review on front end
```
curl -X GET "https://book-api-968113828557.us-central1.run.app//books?title=Coraline" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"

```

# pivoting to using Cloud Composer instaed of Vertex AI

1. take Google Colab book api code and put it in file, adapting to use bucket. fetch_books_dag.py
```
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta
import requests
import csv
import time
import os
from google.cloud import storage

API_KEY = "AIzaSyAF9e-dplvn7hy3ObmK60XV-cpht4pMeeY"
GRADE_QUERIES = {
    "K": "kindergarten books",
    "1": "grade 1 books",
    "2": "grade 2 books",
    "3": "grade 3 books",
    "4": "grade 4 books",
    "5": "grade 5 books",
    "6": "grade 6 books",
    "7": "grade 7 books",
    "8": "grade 8 books"
}

BUCKET_NAME = "k8-books-bucket"
EXISTING_FILE = "books.csv"
NEW_FILE = "k8_books_new.csv"


def download_existing_books():
    client = storage.Client()
    bucket = client.bucket(BUCKET_NAME)
    blob = bucket.blob(EXISTING_FILE)
    blob.download_to_filename(EXISTING_FILE)
    print(f"Downloaded {EXISTING_FILE} from {BUCKET_NAME}")

def upload_new_books():
    client = storage.Client()
    bucket = client.bucket(BUCKET_NAME)
    blob = bucket.blob(NEW_FILE)
    blob.upload_from_filename(NEW_FILE)
    print(f"Uploaded {NEW_FILE} to {BUCKET_NAME}")

def fetch_books():
    download_existing_books()

    existing_identifiers = set()
    with open(EXISTING_FILE, mode='r', encoding='utf-8') as file:
        reader = csv.DictReader(file)
        for row in reader:
            title = row.get("title", "").strip().lower()
            isbn_10 = row.get("isbn_10", "").strip()
            isbn_13 = row.get("isbn_13", "").strip()
            if title:
                existing_identifiers.add(title)
            if isbn_10:
                existing_identifiers.add(isbn_10)
            if isbn_13:
                existing_identifiers.add(isbn_13)

    def fetch_books_for_grade(query, grade):
        book_data = []
        url = (
            f"https://www.googleapis.com/books/v1/volumes?"
            f"q={query}&maxResults=40&printType=books&langRestrict=en&key={API_KEY}"
        )
        response = requests.get(url)
        books = response.json().get("items", [])

        for book in books:
            info = book.get("volumeInfo", {})
            sale = book.get("saleInfo", {})
            access = book.get("accessInfo", {})

            title = info.get("title", "").strip()
            title_key = title.lower()

            isbn_10 = ""
            isbn_13 = ""
            for identifier in info.get("industryIdentifiers", []):
                if identifier["type"] == "ISBN_10":
                    isbn_10 = identifier["identifier"]
                elif identifier["type"] == "ISBN_13":
                    isbn_13 = identifier["identifier"]

            if (
                title_key in existing_identifiers or
                (isbn_10 and isbn_10 in existing_identifiers) or
                (isbn_13 and isbn_13 in existing_identifiers)
            ):
                continue

            authors = ", ".join(info.get("authors", []))
            publisher = info.get("publisher", "")
            published_date = info.get("publishedDate", "")
            description = info.get("description", "")
            page_count = info.get("pageCount", "")
            categories = ", ".join(info.get("categories", []))
            language = info.get("language", "")
            reading_mode_text = info.get("readingModes", {}).get("text", "")
            reading_mode_image = info.get("readingModes", {}).get("image", "")
            image_small = info.get("imageLinks", {}).get("smallThumbnail", "")
            image_large = info.get("imageLinks", {}).get("thumbnail", "")
            country = sale.get("country", "")
            list_price_amount = sale.get("listPrice", {}).get("amount", "")
            list_price_currency = sale.get("listPrice", {}).get("currencyCode", "")
            buy_link = sale.get("buyLink", "")
            web_reader_link = access.get("webReaderLink", "")
            embeddable = access.get("embeddable", "")

            book_data.append([
                title, authors, publisher, published_date, description,
                isbn_10, isbn_13, reading_mode_text, reading_mode_image,
                page_count, categories, maturity_rating, image_small,
                image_large, language, country, list_price_amount,
                list_price_currency, buy_link, web_reader_link,
                embeddable, grade
            ])

            existing_identifiers.update([title_key, isbn_10, isbn_13])

        return book_data

    all_new_books = []
    for grade, query in GRADE_QUERIES.items():
        print(f"Searching for Grade {grade} books...")
        all_new_books.extend(fetch_books_for_grade(query, grade))
        time.sleep(10)  # Throttle to avoid hitting API limits

    headers = [
        "title", "authors", "publisher", "published_date", "description",
        "isbn_10", "isbn_13", "reading_mode_text", "reading_mode_image",
        "page_count", "categories", "image_small",
        "image_large", "language", "sale_country", "list_price_amount",
        "list_price_currency", "buy_link", "web_reader_link", "embeddable", "grade"
    ]

    with open(NEW_FILE, mode='w', newline='', encoding='utf-8') as file:
        writer = csv.writer(file)
        writer.writerow(headers)
        writer.writerows(all_new_books)

    print(f"Saved {len(all_new_books)} books to {NEW_FILE}")
    upload_new_books()


# Define the DAG
with DAG(
    "fetch_books_pipeline",
    schedule_interval="@daily",
    start_date=datetime(2024, 1, 1),
    catchup=False,
    default_args={"retries": 1, "retry_delay": timedelta(minutes=5)}
) as dag:
    fetch_books_task = PythonOperator(
        task_id="fetch_books",
        python_callable=fetch_books
    )

```

2. grant role in shell
```
gcloud projects add-iam-policy-binding book-recommendations-456120 \
  --member="serviceAccount:service-968113828557@cloudcomposer-accounts.iam.gserviceaccount.com" \
  --role="roles/composer.ServiceAgentV2Ext"
```

3. create Composer environment
```
# takes awhile to work (up to 30 min ish)
gcloud composer environments create book-data-pipeline \
  --location=us-central1 \
  --image-version=composer-2.12.0-airflow-2.10.2 \
  --environment-size=small
```

4. when done, find bucket name
```
gcloud composer environments describe book-data-pipeline \
  --location=us-central1 \
  --format="value(config.dagGcsPrefix)"

# example output (use in next line):
# gs://us-central1-book-data-pipel-122da488-bucket/dags
```

5. upload DAG to Composer
```
gsutil cp fetch_books_dag.py gs://us-central1-book-data-pipel-122da488-bucket/dags
```

6. Run DAG: search Cloud Composer >> Environment >> select the right environemnt >> click AirFlow UI >> find your DAG (takes a coupke minutes to show up >> make sure toggled on >> click play button to run

7. clean up the csv by creating file clean_books_csv.py
```
import csv
from google.cloud import storage

BUCKET_NAME = "k8-books-bucket"
SOURCE_FILE = "k8_books_new.csv"
TARGET_FILE = "k8_books_clean.csv"

# Expected columns (22)
COLUMNS = [
    "title", "authors", "publisher", "published_date", "description",
    "isbn_10", "isbn_13", "reading_mode_text", "reading_mode_image", "page_count",
    "categories", "image_small", "image_large", "language",
    "sale_country", "list_price_amount", "list_price_currency",
    "buy_link", "web_reader_link", "embeddable", "grade"
]

client = storage.Client()
bucket = client.bucket(BUCKET_NAME)

# Download the file
blob = bucket.blob(SOURCE_FILE)
blob.download_to_filename("temp_books.csv")

# Clean and save locally
with open("temp_books.csv", "r", encoding="utf-8") as infile, open("k8_books_clean.csv", "w", encoding="utf-8", newline='') as outfile:
    reader = csv.reader(infile)
    writer = csv.writer(outfile)
    writer.writerow(COLUMNS)

    for row in reader:
        if len(row) == 23:  # has 'id' column
            row = row[1:]
        if len(row) == 22 and row[0].strip():  # must have a title
            writer.writerow(row)

# Upload cleaned version
clean_blob = bucket.blob("k8_books_clean.csv")
clean_blob.upload_from_filename("k8_books_clean.csv")

print("Cleaned file uploaded as k8_books_clean.csv")
```

```
# in shell
python3 clean_books_csv.py

# confirm in bucket
gsutil ls gs://k8-books-bucket/k8_books_clean.csv
```

8.  enable service agent
```
gcloud services enable sqladmin.googleapis.com

# create an agent
gcloud iam service-accounts create cloudsql-uploader \
  --description="Used by Cloud SQL to read CSVs" \
  --display-name="Cloud SQL Uploader"

# grant permission
gsutil iam ch \
  serviceAccount:cloudsql-uploader@book-recommendations-456120.iam.gserviceaccount.com:objectViewer \
  gs://k8-books-bucket

```

9.  import to SQL
```
gcloud sql import csv book-recs-db \
  gs://k8-books-bucket/k8_books_clean.csv \
  --database=book_recommendations_d \
  --table=books \
  --columns="title,authors,publisher,published_date,description,isbn_10,isbn_13,reading_mode_text,reading_mode_image,page_count,categories,image_small,image_large,language,sale_country,list_price_amount,list_price_currency,buy_link,web_reader_link,embeddable,grade" \
  --project=book-recommendations-456120
```
10.  fixing issue
```
gsutil cp gs://k8-books-bucket/k8_books_clean.csv .
awk 'BEGIN{FS=OFS=","} NR==1{$1="id,"$1} NR>1{$1=","$1} 1' k8_books_clean.csv > k8_books_with_id.csv

gsutil cp k8_books_with_id.csv gs://k8-books-bucket/

```

# skip merging the tables and go back to fixing the front end
1. fix db.py (host was not connecting to right place)
```
import psycopg2
import os

def get_db_connection():
    return psycopg2.connect(
        dbname=os.environ["DB_NAME"],
        user=os.environ["DB_USER"],
        password=os.environ["DB_PASSWORD"],
        host=f"/cloudsql/{os.environ['INSTANCE_CONNECTION_NAME']}"  # Cloud SQL socket path
    )
```

2. rebuild and push Docker
```
docker build -t gcr.io/book-recommendations-456120/book-api .
docker push gcr.io/book-recommendations-456120/book-api
```

3. redeploy to Cloud Run
```
gcloud run deploy book-api \
  --image gcr.io/book-recommendations-456120/book-api \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --add-cloudsql-instances=book-recommendations-456120:us-central1:book-recs-db \
  --set-env-vars DB_NAME=book_recommendations_db,DB_USER=postgres,DB_PASSWORD=stimac-cis655-final,INSTANCE_CONNECTION_NAME=book-recommendations-456120:us-central1:book-recs-db \
  --port 8080
```
4. YAY it works now

5. 
