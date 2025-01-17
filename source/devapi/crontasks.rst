Automatic actions
-----------------

Goals
^^^^^

Provide a scheduler for background tasks used by GLPI and its plugins.

Implementation overview
^^^^^^^^^^^^^^^^^^^^^^^

The entry point of automatic actions is the file ``front/cron.php``. On each execution, it executes a limited number of automatic actions.

There are two ways to wake up the scheduler :
 - when a user browses in GLPI (the internal mode)
 - when the operating system's scheduler calls ``front/cron.php`` (the external mode)

When GLPI generates an HTML page for a browser, it adds an invisible image generated by ``front/cron.php``. This way, the automatic action runs in a separate process and does not impact the user.

The automatic actions are defined by the ``CronTask`` class. GLPI defines a lot of them for its own needs. They are created in the installation or upgrade process.

Implementation
^^^^^^^^^^^^^^

Automatic actions could be related to an itemtype and the implemention is defined in its class or haven't any itemtype relation and are implemented directly into ``CronTask`` class.

When GLPI shows a list of automatic actions, it shows a short description for each item. The description is gathered in the static method ``cronInfo()`` of the itemtype.

.. Note::

   An itemtype may contain several automatic actions.

Example of implementation from the ``QueuedNotification``:

.. code-block:: php

   <?php
   class QueuedNotification extends CommonDBTM {

      // ...

      /**
       * Give cron information
       *
       * @param $name : automatic action's name
       *
       * @return arrray of information
      **/
      static function cronInfo($name) {

         switch ($name) {
            case 'queuednotification' :
               return array('description' => __('Send mails in queue'),
                            'parameter'   => __('Maximum emails to send at once'));
         }
         return [];
      }

      /**
       * Cron action on notification queue: send notifications in queue
       *
       * @param CommonDBTM $task for log (default NULL)
       *
       * @return integer either 0 or 1
      **/
      static function cronQueuedNotification($task=NULL) {
         global $DB, $CFG_GLPI;

         if (!$CFG_GLPI["notifications_mailing"]) {
            return 0;
         }
         $cron_status = 0;

         // Send mail at least 1 minute after adding in queue to be sure that process on it is finished
         $send_time = date("Y-m-d H:i:s", strtotime("+1 minutes"));

         $mail = new self();
         $pendings = self::getPendings(
            $send_time,
            $task->fields['param']
         );

         foreach ($pendings as $mode => $data) {
            $eventclass = 'NotificationEvent' . ucfirst($mode);
            $conf = Notification_NotificationTemplate::getMode($mode);
            if ($conf['from'] != 'core') {
               $eventclass = 'Plugin' . ucfirst($conf['from']) . $eventclass;
            }
   
            $result = $eventclass::send($data);
            if ($result !== false && count($result)) {
               $cron_status = 1;
               if (!is_null($task)) {
                  $task->addVolume($result);
               }
            }
         }

         return $cron_status;
      }

      // ...

   }

If the argument ``$task`` is a ``CronTask`` object, the method must increment the quantity of actions done. In this example, each notification type reports the wuantity of notification processed and is added to the task's volume.

Register an automatic actions
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Automatic actions are defined in the empty schema located in ``install/mysql/``. Use  the existing sql queries creating rows in the table ``glpi_crontasks`` as template for a new automatic action.

To handle upgrade from a previous version, the new automatic actions must be added in the appropriate update file ``install/update_xx_to_yy.php``.

.. code-block:: php

   <?php
   // Register an automatic action
   CronTask::register('QueuedNotification', 'QueuedNotification', MINUTE_TIMESTAMP,
         array(
         'comment'   => '',
         'mode'      => CronTask::MODE_EXTERNAL
   ));

The ``register`` method takes four arguments:

* ``itemtype``: a ``string`` containing an itemtype name containing the automatic action implementation
* ``name``: a ``string`` containing the name of the automatic action
* ``frequency`` the period of time between two executions in seconds (see ``inc/define.php`` for convenient constants)
* ``options`` an array of options

.. Note::

   The name of an automatic action is actually the method's name without the prefix cron. In the example, the method ``cronQueuedNotification`` implements the automatic action named ``QueuedNotification``.
