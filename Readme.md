<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#orgfa26d24">Automatic sorting of items in org mode</a>
<ul>
<li><a href="#orgab593c6">Motivation</a></li>
<li><a href="#orgb3c9acd">Overview</a></li>
<li><a href="#org4703e0d">Configuration</a></li>
<li><a href="#org76c323b">Defaults</a></li>
<li><a href="#org931976d">Implementation</a>
<ul>
<li><a href="#orgb403e90">Header</a></li>
<li><a href="#org6f28c00">Custom variables</a></li>
<li><a href="#orge6485e4">Standard sorting functions</a></li>
<li><a href="#org7749cd9">General sorting routine</a></li>
<li><a href="#orgd9fa159">File epilogue</a></li>
</ul>
</li>
<li><a href="#org53d0108">Ideas</a>
<ul>
<li><a href="#org25a0fad">Sort items when opening org file, on edit??</a></li>
<li><a href="#orged315c3">do not use org-sort, because it does not allow to combine sorts (i.e. sort by one criteria, if equal - by other)</a></li>
<li><a href="#orgd8d9485">allow to define sort criteria like a lisp function in the properties field</a></li>
<li><a href="#org149b11c">Do not sort only but filter items in org files/agenda</a></li>
<li><a href="#org6ee0773">Take care about exact position for <code>C-c C-c</code> (say, we are inside the table - user may not want to sort)</a></li>
<li><a href="#org7963246">Sort only items, matching org search regex</a></li>
<li><a href="#org4cfe334">Handle nothing to sort</a></li>
<li><a href="#org34b8873">make interactive versions of sorting functions</a></li>
<li><a href="#org7a7842c">autosort - do not sort but show agenda</a></li>
<li><a href="#org739c797">add hooks to to autosort</a></li>
<li><a href="#org6c0c82d">auto add hooks according to the sort type - should be able to define hooks for every sort type</a></li>
<li><a href="#org511b060">get rid of annoying unfolding after <code>org-sort</code></a></li>
<li><a href="#org3963992">put buffer name in error report for wrong element of sorting strategy</a></li>
<li><a href="#orga8644e5">should be able to define alias in sorting strategy</a></li>
<li><a href="#org47d3284">rewrite sorting strategy to use assoc lists</a></li>
<li><a href="#org2afbd84">use local hook in autosort for toggle hooks</a></li>
<li><a href="#org4af49fe">add this functionality? Sorting Org Mode lists using a sequence of regular expressions  13</a></li>
<li><a href="#orgbb247a6">do not raise error but put a message and do not sort on wrong :SORTING: format</a></li>
</ul>
</li>
</ul>
</li>
</ul>
</div>
</div>


<a id="orgfa26d24"></a>

# Automatic sorting of items in org mode


<a id="orgab593c6"></a>

## Motivation

For now, standard org capabilities allow to sort entries in the agenda
views automatically. However, sorting of the actual org files is
rather limited. We only have `org-sort` function, which allows to sort
by one criteria only (not like `org-agenda-sorting-strategy` with
multiple sort criteria). Moreover, it can be done manually only.  

Sometimes, one may have a need to look at some kind of long list of
TODO items (for example, this infinitely growing list of things you
wish to do one day, or list of ideas). It is not comfortable to scroll
across such kind of list to look for specific item. Sorting is
something, that can help here.

<div class="inlinetask">
<b>Add some example here</b><br />
nil</div>

Of course, you can still use `org-sort` or agenda view with restriction
to the current subtree, but it may be disrupting if you want to look
through multiple of such a lists.  

The solution is to implement automatic sorting of subtrees in org
files.


<a id="orgb3c9acd"></a>

## Overview

This package aims to implement an automatic sorting of the subtrees
in org files. The sorting order can be set globally through all the
org files, locally in file, or locally in a subtree using `:SORT:`
property.  

Everything, except global sorting order, can be set using standard
inheritance capabilities of the org properties (file local, subtree
local with or without inheritance for subtrees inside the
subtree). Global sorting order can be set via
`org-autosort-global-sorting-strategy` variable.


<a id="org4703e0d"></a>

## Configuration

Both `:SORT:` property and `org-autosort-global-sorting-strategy`
are lists, which determine how to sort the entries.

<a id="orga79d54d"></a>
`org-autosort-global-sorting-strategy` defined how to sort entries by
default. It is a list of [sorting rules](#org35fb973), defining the comparison
between sorted entries. First, the entries are sorted via first rule
from the list. If the calculated keys are equal, second rule is used,
and so on.

`:SORT:` can be either `nil`, `t`, or the same format as
`org-autosort-global-sorting-strategy`:

-   **`t`:** use `org-autosort-global-sorting-strategy`
-   **not defined or `nil`:** sort if `org-autosort-sort-all` is non `nil`
-   **empty or `none`:** do not sort entries
-   **list:** define separate sorting strategy

The sorting can be done after:

-   opening the org file
-   `org-autosort-sort-entries-at-point` command
-   `org-autosort-sort-entries-in-file` command

(see [sorting triggers](#orga6cfe20) for details)


<a id="org76c323b"></a>

## Defaults

The package provide some predefined sorting rules <a id="org35fb973"></a>,
all are listed in `org-autosort-functions-alist`.

    (defcustom org-autosort-functions-alist '((todo-up-0 . (:key org-autosort-get-todo :cmp <)) ; default org-sort comparison
    					  (todo-down-0 . (:key org-autosort-get-todo :cmp >))
    					  ;; compare according to `org-autosort-todo-cmp-order'
    					  (todo-up . (:key org-get-todo-state :cmp org-autosort-custom-cmp-todo))
    					  (todo-down . (:key org-get-todo-state :cmp (lambda (a b)
    										       (not (org-autosort-custom-cmp-todo a b)))))
    					  ;;					  
    					  (text-up . (:key org-autosort-get:cmp :cmp string<))
    					  (text-down . (:key org-autosort-get-text :cmp string>))
                                              (priority-up . (:key (org-autosort-get-property "PRIORITY") :cmp string<))
                                              (priority-down . (:key (org-autosort-get-property "PRIORITY") :cmp string>)))
      "Alist, defining aliases to sorting rules.
    Each value in the list defines a sorting rule.
    The rule is a property list with :key and :cmp properties.
    
    :key property defines a function to calculate the key value.
    :cmp property defines a function to compare the keys.
    In both cases, function can be defined as
     1. lambda expression
     2. function symbol
     3. list, containing function symbol or lambda expression and their arguments
    
    :key function is called with pos at the entry, without arguments.
    If :key is defined as in 3, all the nesessary arguments should be in the list.
    
    :cmp function must accept two arguments (after all the arguments as in 3).
    It must satisfy the rules of cmp function for `sort'.
    If :cmp is omitted, `org-autosort-default-cmp-function' is used."
      :type '(alist :key-type symbol
    		:value-type (plist :value-type (choise function
    						       (list function (repeat sexp))))))
    
    (defcustom org-autosort-default-cmp-function #'string<
      "Default function, used to compare two entry keys.
    Can be also a list of function and its arguments.
    It is used if cmp function is not defined.
    It must accept two arguments - first and second sorting key to compare.
    Non nil return value means that first key is lesser than second key."
      :type '(function))

You can control automatic sorting by setting <a id="orga6cfe20"></a>

    (defcustom org-autosort-sort-at-file-open t
      "Non nil states for sorting of all items in the org file after opening."
      :type '(boolean))

Default [sorting strategy](#orga79d54d) is

    (defcustom org-autosort-global-sorting-strategy '(priority-down todo-up)
      "Sorting strategy, used to sort entries with :SORT: property not set or nil.
    This is a list, which elements can be:
    - key of the sorting rule from `org-autosort-functions-alist'
    - sorting rule, defined as in `org-autosort-functions-alist'
    - :key values as from `org-autosort-functions-alist'
    Sorting rules are applied accorting the their position in the list.
    nil means that no sorting should be done by default."
      :type '(choice symbol
    		 (plist :value-type (choise function
    					    (list function (repeat sexp))))))


<a id="org931976d"></a>

## Implementation


<a id="orgb403e90"></a>

### Header

    ;;; org-autosort.el --- Sort entries in org files automatically -*- lexical-binding: t; -*-
    
    ;; Version: 0.11
    ;; Author: Ihor Radchenko <yantar92@gmail.com>
    ;; Created: 10 Dec 2017
    ;; Keywords: matching, outlines
    ;; Homepage: https://github.com/yantar92/org-autosort
    ;; Package-Requires: (org)
    
    ;;; Commentary:
    
    ;; This package aims to implement an automatic sorting of the subtrees in org files.
    ;; The sorting order can be set globally through all the org files, locally in file, or locally in a subtree using :SORT: property.
    
    ;;; Code:


<a id="org6f28c00"></a>

### Custom variables

    (defgroup org-autosort nil
      "Customization options of org-autosort package.")

-   to sort or not to sort

    (defcustom org-autosort-sort-all nil
      "Sort entries if :SORT: property is not defined.")

-   auto sort triggers

    (defcustom org-autosort-sort-at-file-open t
      "Non nil states for sorting of all items in the org file after opening."
      :type '(boolean))

-   predefined sorts

    (defcustom org-autosort-functions-alist '((todo-up-0 . (:key org-autosort-get-todo :cmp <)) ; default org-sort comparison
    					  (todo-down-0 . (:key org-autosort-get-todo :cmp >))
    					  ;; compare according to `org-autosort-todo-cmp-order'
    					  (todo-up . (:key org-get-todo-state :cmp org-autosort-custom-cmp-todo))
    					  (todo-down . (:key org-get-todo-state :cmp (lambda (a b)
    										       (not (org-autosort-custom-cmp-todo a b)))))
    					  ;;					  
    					  (text-up . (:key org-autosort-get:cmp :cmp string<))
    					  (text-down . (:key org-autosort-get-text :cmp string>))
                                              (priority-up . (:key (org-autosort-get-property "PRIORITY") :cmp string<))
                                              (priority-down . (:key (org-autosort-get-property "PRIORITY") :cmp string>)))
      "Alist, defining aliases to sorting rules.
    Each value in the list defines a sorting rule.
    The rule is a property list with :key and :cmp properties.
    
    :key property defines a function to calculate the key value.
    :cmp property defines a function to compare the keys.
    In both cases, function can be defined as
     1. lambda expression
     2. function symbol
     3. list, containing function symbol or lambda expression and their arguments
    
    :key function is called with pos at the entry, without arguments.
    If :key is defined as in 3, all the nesessary arguments should be in the list.
    
    :cmp function must accept two arguments (after all the arguments as in 3).
    It must satisfy the rules of cmp function for `sort'.
    If :cmp is omitted, `org-autosort-default-cmp-function' is used."
      :type '(alist :key-type symbol
    		:value-type (plist :value-type (choise function
    						       (list function (repeat sexp))))))
    
    (defcustom org-autosort-default-cmp-function #'string<
      "Default function, used to compare two entry keys.
    Can be also a list of function and its arguments.
    It is used if cmp function is not defined.
    It must accept two arguments - first and second sorting key to compare.
    Non nil return value means that first key is lesser than second key."
      :type '(function))

-   default sorting strategy

    (defcustom org-autosort-global-sorting-strategy '(priority-down todo-up)
      "Sorting strategy, used to sort entries with :SORT: property not set or nil.
    This is a list, which elements can be:
    - key of the sorting rule from `org-autosort-functions-alist'
    - sorting rule, defined as in `org-autosort-functions-alist'
    - :key values as from `org-autosort-functions-alist'
    Sorting rules are applied accorting the their position in the list.
    nil means that no sorting should be done by default."
      :type '(choice symbol
    		 (plist :value-type (choise function
    					    (list function (repeat sexp))))))


<a id="orge6485e4"></a>

### Standard sorting functions

-   by property

        (defun org-autosort-get-property (property)
          "Get the value of PROPERTY for sorting."
          (org-entry-get (point)
        		 property
        		 'selective))

-   By todo keyword

        (defun org-autosort-get-todo ()
          "Get the value of todo keyword for sorting." ; stolen from org-sort-entries in org.el
          (let* ((m (org-get-todo-state))
        	 (s (if (member m
        			org-done-keywords) '- '+))
        	 )
            (- 99
               (funcall s
        		(length (member m
        				org-todo-keywords-1))))))

-   By todo keyword, custom

        (defvar org-autosort-todo-cmp-order nil
          "Order of todo keywords to be shown in sorted subtrees.
               Follow `org-todo-keywords-1' if nil."
          )
        (defun org-autosort-custom-cmp-todo (a b)
          "Compare todo keywords A and B.  Return non nil if A<B."
          (let* ((todo-cmp-orgder (or org-autosort-todo-cmp-order
        			      org-todo-keywords-1))
        	 (posa (or (seq-position org-autosort-todo-cmp-order
        				 a)
        		   0))
        	 (posb (or (seq-position org-autosort-todo-cmp-order
        				 b)
        		   0)))
            (< posa posb)))

-   Alphabetic

        (defun org-autosort-get-text ()
          "Get the text or tags (if text is empty) of the current entry."
          (nth 4 (org-heading-components))
          )

-   By first inactive timestamp

-   By deadline

-   By clocking time

-   Numerically, beginning of the entry/item

-   By scheduled time/date

-   By active timestamp

-   By any timestamp


<a id="org7749cd9"></a>

### General sorting routine

    (defun org-autosort-sorting-strategy-elementp (elm)
      "Validate element ELM of sorting strategy.  Return (:key ... [:cmp ...]) if element and nil otherwise."
      (pcase elm
        (`(quote val)
         (org-autosort-sorting-strategy-elementp val))
        ((pred functionp)
         (list :key elm))
        ((pred (lambda (arg) (assoc arg org-autosort-functions-alist)))
         (alist-get elm org-autosort-functions-alist))
        ((pred (lambda (arg) (plist-get arg :key)))
         (let ((key (org-autosort-sorting-strategy-elementp (plist-get elm :key)))
    	   (cmp (org-autosort-sorting-strategy-elementp (plist-get elm :cmp))))
           (cond ((and key (not cmp)) key)
    	     ((and key cmp) (plist-put key :cmp (plist-get cmp :key)))
    	     (t nil))))
        (`(,func . ,args)
         (if (functionp func)
    	 (list :key elm)
           nil))
        (_ nil)))
    
    (defun org-autosort-sorting-strategyp (sorting-strategy)
      "Validate if SORTING-STRATEGY is a valid and return it.
    The strategy is ensured to be a list.
    Signal user error and return nil if argument is not a sorting strategy."
      (if (not sorting-strategy)
          nil
        (or (let ((res (org-autosort-sorting-strategy-elementp sorting-strategy)))
    	  (if res (list res)))
    	(let* ((testresult (mapcar (lambda (elm) (cons (org-autosort-sorting-strategy-elementp elm)
    						  elm))
    				   sorting-strategy))
    	       (err-elm (alist-get nil testresult 'none)))
    	  (if (equal err-elm 'none)
    	      sorting-strategy
    	    nil
    	    (user-error "Wrong element of sorting strategy: \"%s\" in buffer: %s"
    			err-elm (buffer-name)))))))
    
    (defun org-autosort-get-sorting-strategy ()
      "Get sorting strategy at point for the current entry's subtree being sorted."
      (let ((property (org-entry-get (point) "SORT" 'selective)))
        (pcase property
          ('t (org-autosort-sorting-strategyp org-autosort-global-sorting-strategy))
          ('nil (and org-autosort-sort-all
    	       (org-autosort-sorting-strategyp org-autosort-global-sorting-strategy)))
          ("" nil)
          ('none nil)
          (_ (if (= (cdr (read-from-string property))
    		(length property))
    	     (org-autosort-sorting-strategyp (car (read-from-string property)))
    	   (user-error "Cannot read :SORT: property: \"%s\" in buffer: %s" property (buffer-name))
    	   nil)))))
    
    (defun org-autosort-construct-get-value-function-atom (sorting-strategy-elm)
      "Construct get-value function for single element of sorting strategy (SORTING-STRATEGY-ELM)."
      (let ((key (plist-get (org-autosort-sorting-strategy-elementp sorting-strategy-elm) :key)))
        (pcase key
          ((pred functionp)
           key)
          (`(,func . ,args)
           (when (functionp func)
    	 (lambda () (apply (car key) (cdr key)))))
          ('nil (lambda () nil)))))
    
    (defun org-autosort-construct-get-value-function ()
      "Return get-value function at point.
    This function returns a list of sorting keys."
      (let ((sorting-strategy (org-autosort-get-sorting-strategy)))
        (if sorting-strategy
    	(let ((func-list (mapcar #'org-autosort-construct-get-value-function-atom sorting-strategy)))
    	  (lambda () (mapcar #'funcall func-list)))
          (lambda () (list nil)))))
    
    (defun org-autosort-construct-cmp-function-atom (sorting-strategy-elm)
      "Construct cmp function for single element of sorting strategy (SORTING-STRATEGY-ELM)."
      (let* ((sorting-strategy-elm (org-autosort-sorting-strategy-elementp sorting-strategy-elm))
    	 (cmp (and sorting-strategy-elm
    		   (or (plist-get sorting-strategy-elm :cmp)
    		       org-autosort-default-cmp-function))))
        (pcase cmp
          ((pred functionp)
           (lambda (a b) (funcall cmp a b)))
          (`(,func . ,args)
           (when (functionp func)
    	 (lambda (a b) (apply func a b args))))
          ('nil (lambda (a b) nil)))))
    
    (defun org-autosort-construct-cmp-function ()
      "Return cmp function at point."
      (let ((sorting-strategy (org-autosort-get-sorting-strategy)))
        (if (not sorting-strategy)
    	(lambda (lista listb) nil)
          (let ((cmp-func-list (mapcar #'org-autosort-construct-cmp-function-atom sorting-strategy)))
    	(lambda (lista listb)
    	  (let ((resultlist (seq-mapn (lambda (func a b)
    					(cons (funcall func a b)
    					      (funcall func b a)))
    				      cmp-func-list lista listb)) ; list of cons (a<b . b<a)
    		(done nil)
    		result)
    	    (while (and (not done)
    			(not (seq-empty-p resultlist)))
    	      (let ((elem (pop resultlist)))
    		(unless (and (car elem)
    			   (cdr elem)) ; not equal
    		  (setq done t)
    		  (setq result (car elem)))))
    	    result))))))
    
    (defun org-autosort-org-sort-entries-wrapper (&rest args)
      "Run `org-sort-entries' at point with ARGS if nesessary.
    Make sure, folding state is not changed."
      (when (org-autosort-get-sorting-strategy)
        (save-excursion
          (save-restriction
    	(condition-case err
    	    (apply #'org-sort-entries args)
    	  (user-error
    	   (unless (string-match-p "Nothing to sort"
    				   (error-message-string err))
    	     (signal (car err) (cdr err)))))))))
    
    (defun org-autosort-sort-entries-at-point-nonrecursive ()
      "Sort org-entries at point nonrecursively."
      (interactive)
      (funcall #'org-autosort-org-sort-entries-wrapper
    	   nil ?f
    	   (org-autosort-construct-get-value-function)
    	   (org-autosort-construct-cmp-function)))
    
    (defun org-autosort-sort-entries-at-point-recursive ()
      "Sort org-entries at point recursively."
      (interactive)
      (condition-case err
          (org-map-entries (lambda nil (funcall #'org-autosort-org-sort-entries-wrapper
    				     nil ?f
    				     (org-autosort-construct-get-value-function)
    				     (org-autosort-construct-cmp-function)))
    		       nil 'tree)
        (error
         (if (string-match-p "Before first headline at position"
    			 (error-message-string err))
    	 (org-map-entries (lambda nil (funcall #'org-autosort-org-sort-entries-wrapper
    					nil ?f
    					(org-autosort-construct-get-value-function)
    					(org-autosort-construct-cmp-function)))
    			  nil 'file)
           (signal (car err) (cdr err))))))
    
    (defun org-autosort-sort-entries-at-point (&optional ARG)
      "Sort org entries at point.
    Sort recursively if invoked with \\[universal-argument]."
      (interactive "P")
      (if (equal ARG '(4))
          (org-autosort-sort-entries-at-point-recursive)
        (org-autosort-sort-entries-at-point-nonrecursive)))
    
    (defun org-autosort-sort-entries-in-file ()
      "Sort all entries in the file recursively."
      (interactive)
      (org-map-entries (lambda nil (funcall #'org-autosort-org-sort-entries-wrapper
    				 nil ?f
    				 (org-autosort-construct-get-value-function)
    				 (org-autosort-construct-cmp-function)))
    		   nil 'file))
    
    (defun org-autosort-sort-entries-in-file-maybe ()
      "Sort all entries in the file recursively if `org-autosort-sort-at-file-open' is not nil."
      (when org-autosort-sort-at-file-open (org-autosort-sort-entries-in-file)))
    
    (add-hook 'org-mode-hook #'org-autosort-sort-entries-in-file-maybe)


<a id="orgd9fa159"></a>

### File epilogue

    (provide 'org-autosort)
    
    ;;; org-autosort.el ends here


<a id="org53d0108"></a>

## Ideas


<a id="org25a0fad"></a>

### Sort items when opening org file, on edit??


<a id="orged315c3"></a>

### do not use org-sort, because it does not allow to combine sorts (i.e. sort by one criteria, if equal - by other)


<a id="orgd8d9485"></a>

### allow to define sort criteria like a lisp function in the properties field


<a id="org149b11c"></a>

### Do not sort only but filter items in org files/agenda


<a id="org6ee0773"></a>

### Take care about exact position for `C-c C-c` (say, we are inside the table - user may not want to sort)


<a id="org7963246"></a>

### Sort only items, matching org search regex


<a id="org4cfe334"></a>

### Handle nothing to sort


<a id="org34b8873"></a>

### make interactive versions of sorting functions


<a id="org7a7842c"></a>

### autosort - do not sort but show agenda


<a id="org739c797"></a>

### add hooks to to autosort


<a id="org6c0c82d"></a>

### auto add hooks according to the sort type - should be able to define hooks for every sort type


<a id="org511b060"></a>

### get rid of annoying unfolding after `org-sort`


<a id="org3963992"></a>

### put buffer name in error report for wrong element of sorting strategy


<a id="orga8644e5"></a>

### should be able to define alias in sorting strategy


<a id="org47d3284"></a>

### rewrite sorting strategy to use assoc lists


<a id="org2afbd84"></a>

### use local hook in autosort for toggle hooks


<a id="org4af49fe"></a>

### add this functionality? [Sorting Org Mode lists using a sequence of regular expressions  13](http://sachachua.com/blog/2017/12/sorting-org-mode-lists-using-a-sequence-of-regular-expressions/)


<a id="orgbb247a6"></a>

### do not raise error but put a message and do not sort on wrong :SORTING: format

