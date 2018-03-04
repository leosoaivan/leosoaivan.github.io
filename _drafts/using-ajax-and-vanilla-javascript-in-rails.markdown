
---
layout: post
title: "Deleting a Rails Active Records Object with AJAX & Vanilla JS"
date: 03 March 2018
tags: rails ajax vanilla javascript
---

One of my commitments for this year is to learn more JavaScript, including: a) as a language, fundamentally, b) as a framework (i.e. VueJS), and c) as a part of Ruby on Rails.

*I always knew I was good at juggling.*

The upside of learning Ruby on Rails is that you don't need an deep understanding of JavaScript (or even Ruby, for that matter) to get up and running. It's entirely possible to spend months and months focused exclusively on practicing and writing good, clean Rails code. Unfortunately, therein also lies the downside: while I've become more and more comfortable with Rails, I know nothing of JS beyond the basics.

Having decided that that needed to change, I wanted to see whether or not I could manipulate Rails Active Record objects using AJAX and vanilla JavaScript. I made it a particular point not to use jQuery, as everything jQuery does can be done with vanilla Javascript, and without the weight.

In this particular case, my aim was to delete an instance of a Contact, which belong to an Episode.

# The Test
Because I'm not yet familiar with JavaScript testing, I relied on my current integration testing suite, which utilizes RSpec, Capybara, and Selenium. A valid contact is already available via a `let!` statement.

```ruby
# app/spec/features/managing_contacts_spec.rb

let! (:contact) { #code to create contact }

describe 'deleting a contact', js: true do
  before :each  do
    accept_confirm do
      click_on 'Delete contact'
    end
  end
  
  it 'flashes a successful message' do
    expect(page).to have_css alert_success
  end
  
  it 'does not display the contact' do
    expect(page).not_to have_content contact.body
  end
end
```
# The View
The first order of business was changing the link that a user clicks to delete a contact. Adding `remote: true` is all it takes to enable Rails to make an AJAX request, rather than following the link in question:

```erb
<tr id="contact-<%= contact.id %>">
  <!-- shortened for brevity -->
  <td>
    <%= link_to "Delete contact", contact_path(contact), method: :delete, remote: true, data: { confirm: "Are you sure you want to delete this contact?" } %>
  </td>
</tr>
```

# The Controller
Next, the controller must be able to handle the AJAX request and respond accordingly. At first, I wrote the controller method for `destroy` to serve HTML responses, which is standard Rails.

Originally:

```ruby
# app/controllers/contacts_controller.rb

def  destroy
  contact =  Contact.find(params[:id])
  
  if contact.destroy
    flash[:success] = "Contact successfully deleted."
    redirect_to episode_path(contact.episode)
  else
    flash.now[:danger] = "There was an error. Please try again."
    redirect_to episode_path(contact.episode)
  end
end
``` 
After:
```ruby
# app/controllers/contacts_controller.rb

def  destroy
  contact = Contact.find(params[:id])
  
  if contact.destroy
    respond_to do |format|
      format.js { flash.now[:success] = "Contact successfully deleted." }
    end
  end
end
```
Adding the ability to respond with JS is simple with `respond_to`.

# The (vanilla) JavaScript
The script needed to do a number of things:
* Select the right contact
* Delete the selected contact
* Display a flash to the user

The last point is important to consider. ActionDispatch::Flash provides a convenient way of sending messages between server requests, but this AJAX call is client-sided. The following is what I came up with:
```javascript
// app/views/contacts/destroy.js.erb

const container = document.querySelector(".container")

function removeRow(){
  const contactRow = document.querySelector("tr#contact-<%= contact.id %>")
  contactRow.parentNode.removeChild(contactRow)
}

function insertFlash(){
  const flash =  "<%= j render partial: 'shared/flash' %>"
  container.insertAdjacentHTML('afterbegin', flash)
}

removeRow();
insertFlash();
```

Let's break it down.

First, because my contacts are displayed as rows in a table, selecting the right one meant selecting the right `<tr>` element. Not a problem, since each row is identified via a unique `#id`. Next, the row can only be deleted in relation to its parent node. The parent node can be returned via `parentNode()`. From the parent node, the relevant child node can then be deleted.

Second, the flash required escaping and rendering the right partial. Thankfully, the values in `flash.now` are immediately available within a request. I was able to escape and render the right partial and use `insertAdjacentHTML()`, which allowed me to insert the text representation of the flash as the first child of my app's container, while converting it to proper HTML.

This was sufficient, until an interesting edge case presented itself.

# A Wild Edge Case Appears!
My contacts controller displays a flash when a contact is created. But what happens when I create a contact, and immediately try to delete it?

TWO FLASHES. But of course. The initial flash persists until a new request is made, and the AJAX I implement doesn't make server requests. What was thus needed was a way to identify if a flash was already present, and, if so, remove/replace it. The end result was as follows:

```js
function editFlash(){
  const oldFlash = document.querySelector("#flash")
  const newFlash = "<%= j render partial: 'shared/flash' %>"

  removeOldFlash(oldFlash)
  insertNewFlash(newFlash)
}

function removeOldFlash(flash){
  if (document.contains(flash)){
    container.removeChild(flash)
  }
}

function insertNewFlash(flash){
  container.insertAdjacentHTML('afterbegin', flash)
}
```
Covering the edge case only required checking if a flash was present, and then remove it if it did.

# Another Test, Just in (edge) Case 
Tests are almost necessary to ensure that efforts to refactor don't break your code. Seeing as I'm a JavaScript beginner, I decided to add a little integration test for now.

Because the integration test, as whole, already creates a contact with a `let!` statement, and because I also needed a flash to appear, I created a new context that would allow me to create a new contact in the test.

```ruby
# app/spec/features/managing_contacts.rb

describe 'deleting a contact', js: true do
  context 'with a flash already present' do
    before :each  do
      click_on 'Create contact'
      fill_in 'Body', with:  'This is a new contact.'
      click_on 'Create Contact'
      
      within("tr#contact-#{contact.id}") do
        accept_confirm do
          click_on 'Delete contact'
        end
      end
    end
    
    it 'flashes only one message at a time' do
      expect(page).to have_selector('#flash', count:  1)
    end
  end

  context 'regardless of flash presence' do
    before :each  do
      accept_confirm do
        click_on 'Delete contact'
      end
    end

    it 'flashes a message' do
      expect(page).to have_content 'Contact successfully deleted.'
    end
    
    it 'does not display the contact' do
      expect(page).not_to have_content contact.body
    end
  end
end
```
With these tests, I feel like I've covered the very basics. At this point, I can rethink my test structure and wording, and can refactor code as necessary.

# Conclusion
We walked through the process of modifying/creating the Rails views, controllers, and scripts necessary to delete an Active Record object.

At the end of the day, this wasn't a bad foray into integrating custom JS into Rails. A number of CSS frameworks, including Bootstrap and Materialize, make use of jQuery snippets to add functionality to their components (drop-downs, navbars), but I'll be looking to continue learning JS as it applies to the front-end. As it stands, it's possible to do with vanilla JS what jQuery does, all without adding weight to a web application.

Thanks for reading!
