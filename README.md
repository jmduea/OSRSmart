# Introduction:

During my time at The Tech Academy, I collaborated with a team of peers to develop an interactive website for managing one's collections of things related to various hobbies, as well as API and Data Scraped content for those hobbies and was built using the Django framework. We worked with a legacy codebase, which provided a valuable learning experience in bug fixing, code cleanup, and implementing requested features. Despite encountering some major changes, we managed to deliver the required product on time by using our available resources effectively. This project gave me insight into how skilled developers utilize available resources to create a high-quality product. As part of the team, I contributed to several back-end stories, of which I am particularly proud. However, since much of the site had already been developed, there were also numerous front-end stories and UX enhancements that required varying levels of effort. Each team member had the opportunity to work on front-end and back-end stories. During the two-week sprint, I also developed additional skills in project management and team programming, which I am confident will be useful in future projects.

# OSRSmart Summary:
OSRSmart is a comprehensive interactive website designed to provide users with in-depth information on all OSRS items in the game. With OSRSmart, users can easily track and manage their favorite items, create an account, and log in to access personalized features.

In addition to the extensive catalogue of OSRS items, OSRSmart also scrapes data from the Old School RuneScape Wiki and utilizes the Old School Runescape Grand Exchange API to provide users with valuable information about each item, such as current pricing and price trends. This information is regularly updated to ensure the accuracy and relevance of the data.

The website's homepage is designed to provide users with quick access to their favorited items, as well as the latest Grand Exchange market indexes. With the site's powerful search functionality, users can effortlessly browse through the entire database of OSRS items and access detailed information on each item.

Overall, OSRSmart offers a convenient and efficient way for OSRS players to keep track of their favorite items and stay informed about the latest market trends.

# CRUD Functionality:
- Creating an item and saving it to the database:

      # View function to create a new item in the database
      def create_item(request):
          form = ItemForm(data=request.POST or None)
          if request.method == 'POST':
              if form.is_valid():  # Validates the form
                  form.save()  # Saves the form data to the database
                  return redirect('OSRSmart_catalogue')  # Redirects to the catalogue page to display
                  # all items with the newly added item
              else:
                  form = ItemForm()  # returns the user to the create page with an empty form to be
                  # filled out
           return render(request, 'OSRSmart/OSRSmart_create.html', {'form': form})

- Displaying all items in the database on a catalogue page with filter options:
    
      def display_catalogue(request):
        item_list = Item.objects.all()

        # Filter items by name, current price range, and members
        name_query = request.GET.get('name')
        if name_query:
            item_list = item_list.filter(name__icontains=name_query)

        price_range_query = request.GET.get('price_range')
        if price_range_query:
            price_range = price_range_query.split(',')
            if len(price_range) == 2:
                lower_price = price_range[0]
                upper_price = price_range[1]
                item_list = item_list.filter(current_price__range=(lower_price, upper_price))

        members_query = request.GET.get('members')
        if members_query:
            item_list = item_list.filter(members=members_query)

        paginator = Paginator(item_list, 10)  # Show 10 items per page
        page = request.GET.get('page')
        try:
            item_list_paginated = paginator.page(page)
        except PageNotAnInteger:
            # If page is not an integer, deliver first page.
            item_list_paginated = paginator.page(1)
        except EmptyPage:
            # If page is out of range, deliver last page of results.
            item_list_paginated = paginator.page(paginator.num_pages)

        content = {'item_list': item_list_paginated}
        return render(request, 'OSRSmart/OSRSmart_catalogue.html', content)
    
 - Displaying the details for a specific item (updated with info scraped from the items wiki page if applicable):

        # View function to display all the database info about a specific item from the catalogue
        def display_details(request, item_id):
          item_id = int(item_id)
          item_get = get_object_or_404(Item, pk=item_id)

          try:
              update_prices(item_id)
              scraped_data = scrape_item_info(item_name=item_get.name)
          except:
              scraped_data = {}

          form = ItemForm()  # create an instance of the ItemForm

          # add the item_get object and the scraped data to the context dictionary
          content = {
              'item_get':     item_get,
              'form':         form,
              'scraped_data': scraped_data
          }

          return render(request, 'OSRSmart/OSRSmart_details.html', content)
    
- Deleting or editing items in the database:

          def edit_form(request, item_id):
          """Renders a new page for input and edit form."""
          item_id = int(item_id)
          item = get_object_or_404(Item, pk=item_id)
          form = ItemForm(request.POST or None, instance=item)
          if request.method == 'POST':
              if form.is_valid():
                  form2 = form.save(commit=False)
                  form2.save()
                  return redirect('OSRSmart_catalogue')
          return render(request, 'OSRSmart/OSRSmart_editform.html', {'form': form, 'item': item})


      def delete_item(request, item_id):
          # Convert the ID to an integer
          item_id = int(item_id)
          # Get the Item object with the given primary key
          item = get_object_or_404(Item, pk=item_id)

          # If the request method is POST, delete the item and redirect to catalogue page
          if request.method == 'POST':
              item.delete()
              return redirect('OSRSmart_catalogue')
          # If the request method is not POST, render the confirmation modal with the item to delete
          else:
              # Define the content dictionary to be passed to the template
              content = {"item": item, "item_id": item_id}
              # Render the confirmation modal template with the content dictionary
              return render(request, "OSRSmart/confirm_delete_modal.html", content)
              
- CRUD functionality in action:              
![alt text](https://github.com/jmduea/OSRSmart/blob/main/CRUDFunctionality.gif "CRUD functionality")
# Web Scraping:
- Scraping Grand Exchange Market Index Info and displaying it on the homepage:
![alt text](https://github.com/jmduea/OSRSmart/blob/main/GrandExchangeIndexes.jpg "Grand Exchange Indexes")

      # View function to render the home page
      def osrsmart_home(request):
          url = 'https://oldschool.runescape.wiki/w/RuneScape:Grand_Exchange_Market_Watch'
          response = requests.get(url)

          # Parse the HTML content using BeautifulSoup
          soup = BeautifulSoup(response.content, 'html.parser')

          # Find the table element and extract its rows
          table = soup.find('table', {'class': 'wikitable'})
          rows = table.tbody.find_all('tr')

          # Create a list of dictionaries to store the data for each card
          cards_data = []

          # Extract the data from each row and add it to the list of cards data
          for row in rows:
              columns = row.find_all('td')
              if columns:
                  name = columns[1].text.strip()
                  as_of_today = columns[2].text.strip()
                  change = columns[3].text.strip()

                  card_data = {
                      'title':       name,
                      'value':       as_of_today,
                      'percentage':  change,
                      'icon_class':  'ni ni-money-coins',  # customize the icon class for each card
                      'color_class': 'bg-gradient-green'  # customize the color class for each card
                      }
                  cards_data.append(card_data)
          favorite_items = []
          if request.user.is_authenticated:
              favorite_items = Favorite.objects.filter(user=request.user).select_related('item')
          items = Item.objects.filter(today_trend='positive').order_by(-F('day30_change'))[:10]
          content = {'items': items, 'cards_data': cards_data, 'favorite_items': favorite_items}
          return render(request, 'OSRSmart/OSRSmart_home.html', content)
          
 - Scraping Item specific info from it's wiki page:
<br>![alt text](https://github.com/jmduea/OSRSmart/blob/main/WikiScrapedInfo.jpg "Wiki Scraped Info")
          
        def scrape_item_info(item_name):
            url = f"https://oldschool.runescape.wiki/w/{item_name.replace(' ', '_')}"
            response = requests.get(url)
            soup = BeautifulSoup(response.content, "html.parser")
            # extract the infobox table
            infobox = soup.find("table", class_="infobox no-parenthesis-style infobox-item")
            if infobox is None:
                print("Infobox not found")
                return {}
            else:
                print(infobox.prettify())
            # extract the other fields from the infobox
            rows = infobox.find_all('tr')

            # Loop through each row and extract the key-value pairs
            data = {}
            for row in rows:
                cells = row.find_all(['th', 'td'])
                if len(cells) == 2:
                    key = cells[0].get_text(strip=True)
                    value = cells[1].get_text(strip=True)
                    data[key] = value

            # return a dictionary containing the scraped data
            return data

# API:
- Current price, as well as trend data for 30 days, 90 days, and 180 days is obtained from the OSRS API and displayed in the details page for the item:
<br>![alt text](https://github.com/jmduea/OSRSmart/blob/main/API.jpg "API Info")

          # The function below updates the prices of items on the website.
          def update_prices(item_id):
              item = get_object_or_404(Item, id=item_id)

              # Attempts to retrieve data from cache. If not found, makes a request to the external API.
              cache_key = f"item_{item.id}_data"
              data = cache.get(cache_key)
              if data is None:
                  url = f"https://services.runescape.com/m=itemdb_oldschool/api/catalogue/detail.json?item={item.id}"
                  headers = {
                      'User-Agent': '@UnknwnEntity#2059'
                      }

                  try:
                      response = requests.get(url, headers=headers)
                      response.raise_for_status()
                      data = response.json()
                      # If successful, caches the response data with a timeout of 2 hours.
                      cache.set(cache_key, data, timeout=7200)
                      print(response.status_code)
                  # If there is an error, it returns an error message.
                  except:
                      raise ValueError('Error updating item price.')

              # Updates the item's attributes with the new data and saves it to the database.
              item.icon = data['item']['icon']
              item.icon_large = data['item']['icon_large']
              item.type = data['item']['type']
              item.type_icon = data['item']['typeIcon']
              item.current_price = data['item']['current']['price']
              item.current_trend = data['item']['current']['trend']
              item.today_price = data['item']['today']['price']
              item.today_trend = data['item']['today']['trend']
              item.day30_change = data['item']['day30']['change']
              item.day30_trend = data['item']['day30']['trend']
              item.day90_change = data['item']['day90']['change']
              item.day90_trend = data['item']['day90']['trend']
              item.day180_change = data['item']['day180']['change']
              item.day180_trend = data['item']['day180']['trend']
              item.save()
              
# Front-End Development:
- Home Dashboard:
<br>![alt text](https://github.com/jmduea/OSRSmart/blob/main/Homepage.jpg "OSRSmart Dashboard")
- Search Functionality:
<br>![alt_text](https://github.com/jmduea/OSRSmart/blob/main/SearchFunction.gif "Search Functionality")
