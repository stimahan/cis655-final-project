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
2. Run the following code in chunks
```
#1 start CloudProxy
./cloud-sql-proxy [PROJECT_ID]:[REGION}:[INSTANCE-NAME] -p 5433

#2 connect to PostgreSQL
psql -h 127.0.0.1 -U postgres -p 5433

#3 create a new database
CREATE DATABASE book_recommendations_db;

#4 connect to the database
\c book_recommendations_db

#5 create a table. You may need to do this line by line. Type each line and hit enter. When finished, close the parentheses (see below) and hit enter.
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


#6 verify table creation
\dt

#7  import data from bucket to sql
gcloud sql import csv your-instance-name gs://your-bucket-name/k8_books_40.csv \
  --database=your-database-name \
  --table=books

#8 create a directory
sudo mkdir /cloudsql
sudo chown $USER /cloudsql

#9 re-run proxy
./cloud-sql-proxy \
  --unix-socket /cloudsql \
  [PROJECT_ID]:[REGION}:[INSTANCE-NAME]

#10 reconnect to database (may new tab or terminal)
psql -h /cloudsql/[PROJECT_ID]:[REGION}:[INSTANCE-NAME] \
     -U postgres -d book_recommendations_db

#11 copy csv and include headers; single line of code
\copy books(title,authors,publisher,published_date,description,isbn_10,isbn_13,reading_mode_text,reading_mode_image,page_count,categories,maturity_rating,image_small,image_large,language,sale_country,list_price_amount,list_price_currency,buy_link,web_reader_link,embeddable,grade) FROM 'k8_books_40.csv' WITH CSV HEADER;

```

# Create Backend API
1. Open Cloudshell terminal
```
#1 create a folder and change to that directory
   mkdir book-api && cd book-api

 #2
   







