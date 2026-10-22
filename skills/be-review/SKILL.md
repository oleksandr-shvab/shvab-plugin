---
name: be-review
description: Makes a code review for a current branch PR for Django project.
disable-model-invocation: true
---

Act like a senior lead python django developer. Make a thorough code review for a current pr, ultrathink.  
Pay extra attention to connected to a pr user flows, business logic, find corner cases.  
Spot code smells and a possibility to use design patterns.  
Enforce widely accepted python and django patterns and best practices.  
How to get code for review - review a code in PR for a current branch.  

**Flow-level impact analysis**  
For the big-picture, cross-flow part of this review, launch the `flows-review` agent via the Agent tool (subagent_type: "shvab:flows-review" — use the type exactly as your Agent tool lists it). It traces the call hierarchy of the changed code and finds logical inconsistencies introduced into connected user flows. Do this first, then perform the line-level and best-practice checks below yourself. Merge the agent's findings with your own into a single report; do not duplicate issues it already raised.  

Other things to spot:  
Naming (should underline why, not what code does).  
Orthogonality, encapsulation (decoupling, SRP) (how will i do this from shell / DRF?).  

**Additional**  
Encapsulation - keep data and functions in a single class. (Thin view, thick models from Two Scoops of Django book).  
Ensure that business rules global validation lives in models, model managers, services layer, not in views, forms.
Never assign value to a model field, call model.save(), form.save() directly from view. Always encapsulate state changing operations into model methods and manager methods.  
Using low-level ORM query methods (filter, order_by etc) directly is an anti-pattern. Do it as QuerySet methods.  
If you need an alternate constructor for model instances, do it as a manager method.  
Any sort of logical operation on only a single model instance should be a method on the model class.  
Any sort of logical operation on a set of instances of the same model should be a method on the model’s manager or its QuerySet.  

Avoid using the Django queryset.update() method, model manager bulk_create() - they skip custom model save(), do not send signals.
Enforce DB normalization. Add model @property or annotated querysets if needed.  
Raise in case of an unexpected flow. Dead programs tell no lies (Pragmatic Programmer book).  
Spot n+1 query issues, but avoid micro-optimizations.  
Check that at least 1 test per endpoint touched in PR is included. Tests are mandatory!

Avoid list(queryset) - it forces queryset evaluation, not resource-optimal.  
Prefer is_ prefix for Values keeping boolean.  
Prevent prints, make sure no %-formatting is used, and modern f-string formatting is used.  
Make sure imports are on top of the file, unless it is needed for a circular dependency fix.  
Prevent excessive usage of javascript in templates. Consider Alpine or HTMX usage instead.  
Do not use 1-symbol or 2-symbol variable names (except of commonly accepted, like df for dataframe).
If method does not accept any args - make it a @property.

Spot magic constant code smell. Use TextChoices / Enum  
