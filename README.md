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

1. start CloudProxy
```
./cloud-sql-proxy [PROJECT_ID]:[REGION}:[INSTANCE-NAME] -p 5433
```

2. connect to PostgreSQL
```
psql -h 127.0.0.1 -U postgres -p 5433
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

5. create a .env file for your db
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

8. run the cloud sql proxy, it should return "Listening on...". If you see this, keep this open and open another tab. You will do this anytime you want to connect the session.
```
./cloud-sql-proxy [PROJECT-ID]:[REGION]:[INSTANCE-ID]
```

9. in the new tab, test your conneciton
```
psql -h /cloudsql/[PROJECT-ID]:[REGION]:[INSTANCE-ID] -U postgres -d book_recommendations_db
```

10. ensure that you are in the right directory and activate the virtual environment (do this every new session)
```
cd ~/book-api
source venv/bin/activate
```

11. Run your app  (do this every time you change app.py or are starting a new session)
```
uvicorn app:app --host=0.0.0.0 --port=8080 --reload
```

12. click the "Web Preview" button and select "preview on port 8080"
13. If this gives an error, click on the url and add /docs to the end and enter
    orginal url: "https://8080-cs-399637968661-default.cs-us-central1-pits.cloudshell.dev/?authuser=0"
    new url: "https://8080-cs-399637968661-default.cs-us-central1-pits.cloudshell.dev/docs"
14. Updated app.py to add ability to search for books by title, author, category, and grade. Also add ability to add, update, and delete books.
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

15.








