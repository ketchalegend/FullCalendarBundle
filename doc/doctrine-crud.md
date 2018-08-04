# Doctrine Basic CRUD example with FullCalendarBundle

Create an entity with at least a `startDate` and an `endDate` you can also add a `title`

For this example we call it `Booking` entity
```php
<?php

namespace AppBundle\Entity;

use Doctrine\ORM\Mapping as ORM;

/**
 * @ORM\Entity()
 */
class Booking
{
    /**
     * @ORM\Id()
     * @ORM\GeneratedValue()
     * @ORM\Column(type="integer")
     */
    private $id;

    /**
     * @ORM\Column(type="datetime")
     */
    private $beginAt;

    /**
     * @ORM\Column(type="datetime")
     */
    private $endAt;

    /**
     * @ORM\Column(type="string", length=255)
     */
    private $title;


    public function getId()
    {
        return $this->id;
    }

    public function getBeginAt()
    {
        return $this->beginAt;
    }

    public function setBeginAt(\DateTime $beginAt)
    {
        $this->beginAt = $beginAt;
    }

    public function getEndAt()
    {
        return $this->endAt;
    }

    public function setEndAt(\DateTime $endAt)
    {
        $this->endAt = $endAt;
    }

    public function getTitle()
    {
        return $this->title;
    }

    public function setTitle(string $title)
    {
        $this->title = $title;
    }
}

```

Then create or generate a CRUD for your entity by running:

```sh
# Symfony 3
php bin/console doctrine:generate:crud 

# Symfony flex (composer req maker)
php bin/console make:crud
```

Create a action and a template in the controller to display the calendar

```php
<?php

namespace AppBundle\Controller;

// use ...

/**
 * @Route("/booking")
 */
class BookingController extends Controller
{
    // ...

    /**
     * @Route("/calendar", name="booking_calendar")
     */
    public function calendar()
    {
        return $this->render('booking/calendar.html.twig');
    }

    // ...
}
```
the calendar template with a link to the new form page:
```twig
{% extends 'base.html.twig' %}

{% block body %}
    <a href="{{ path('booking_new') }}">Create new booking</a>

    {% include '@FullCalendar/Calendar/calendar.html.twig' %}
{% endblock %}

{% block stylesheets %}
    <link rel="stylesheet" type="text/css" href="https://cdnjs.cloudflare.com/ajax/libs/fullcalendar/3.9.0/fullcalendar.min.css">
{% endblock %}

{% block javascripts %}
    <script type="text/javascript" src="https://code.jquery.com/jquery-3.3.1.min.js"></script>
    <script type="text/javascript" src="https://momentjs.com/downloads/moment.min.js"></script>
    <script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/fullcalendar/3.9.0/fullcalendar.min.js"></script>
    <script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/fullcalendar/3.9.0/locale-all.js"></script>

    <script type="text/javascript">
        $(function () {
            $('#calendar-holder').fullCalendar({
                locale: 'fr',
                header: {
                    left: 'prev, next, today',
                    center: 'title',
                    right: 'month, agendaWeek, agendaDay'
                },
                allDaySlot: false,
                lazyFetching: true,
                navLinks: true,
                selectable: true,
                editable: true,
                eventDurationEditable: true,
                eventSources: [
                    {
                        url: "{{ path('fullcalendar_load_events') }}",
                        type: 'POST',
                        data:  {
                            filters: { 'foo': 'bar' }
                        },
                        error: function () {
                            alert('There was an error while fetching FullCalendar!');
                        }
                    }
                ]
            });
        });
    </script>
{% endblock %}

```

We now have to link the CRUD to the calendar

To do this modify the listener to access to the router interface

```php
<?php
namespace AppBundle\EventListener;

// ...
use Symfony\Component\Routing\Generator\UrlGeneratorInterface;

class FullCalendarListener
{
    // ...

    /**
     * @var UrlGeneratorInterface
     */
    private $router;

    public function __construct(EntityManagerInterface $em, UrlGeneratorInterface $router)
    {
        $this->em = $em;
        $this->router = $router;
    }
```

Then use the `setUrl()` function on your `Event` object variable
```php
$bookingEvent->setUrl(
    $this->router->generate('booking_show', array(
        'id' => $booking->getId(),
    ))
);
```

Full listener for `Booking` entity
```php
<?php

namespace AppBundle\EventListener;

use AppBundle\Entity\Booking;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Component\Routing\Generator\UrlGeneratorInterface;
use Toiba\FullCalendarBundle\Entity\Event;
use Toiba\FullCalendarBundle\Event\CalendarEvent;

class FullCalendarListener
{
    /**
     * @var EntityManagerInterface
     */
    private $em;

    /**
     * @var UrlGeneratorInterface
     */
    private $router;

    public function __construct(EntityManagerInterface $em, UrlGeneratorInterface $router)
    {
        $this->em = $em;
        $this->router = $router;
    }

    public function loadEvents(CalendarEvent $calendar)
    {
        $startDate = $calendar->getStart();
        $endDate = $calendar->getEnd();
        $filters = $calendar->getFilters();

        // You may want do a custom query to populate the calendar
        // b.beginAt is the start date in the booking entity
        $bookings = $this->em->getRepository(Booking::class)
            ->createQueryBuilder('b')
            ->andWhere('b.beginAt BETWEEN :startDate and :endDate')
            ->setParameter('startDate', $startDate->format('Y-m-d H:i:s'))
            ->setParameter('endDate', $endDate->format('Y-m-d H:i:s'))
            ->getQuery()->getResult();

        foreach($bookings as $booking) {

            // create an event with the booking data
            $bookingEvent = new Event(
                $booking->getTitle(),
                $booking->getBeginAt(),
                $booking->getEndAt() // If end date is null or not defined, it create an all day event
            );

            /*
             * For more information see : Toiba\FullCalendarBundle\Entity\Event
             * and : https://fullcalendar.io/docs/event-object
             */
            // $bookingEvent->setBackgroundColor($booking['bgColor']);
            // $bookingEvent->setCustomField('borderColor', $booking['bgColor']);

            $bookingEvent->setUrl(
                $this->router->generate('booking_show', array(
                    'id' => $booking->getId(),
                ))
            );

            // finally, add the booking to the CalendarEvent for displaying on the calendar
            $calendar->addEvent($bookingEvent);
        }
    }
}
```

Now in the calendar when we click on an event it show the `showAction()` that contains an edit and delete link

And when you create a new `Booking` (or your custom entity name) it appear on the calendar

Don't forget to [register the listener](index.md#4-create-your-listener) as a service like in the installation process

If you have create a custom entity don't forget to modify listener: 
 - Replace all `Booking` or `booking` by your custom entity name
 - In the query near the `andWhere` modify `beginAt` to your custom start event date attribute
 - Also near the `new Event(` in the `foreach` modify the getters to fit toyour needs and entity