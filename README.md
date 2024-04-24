# Temp-Repo

@login_required
def entry_actions(request):


    action = ''

    req_form = request.GET.get('form','')

    req_user = request.GET.get('user','')



    entries_obj = Entries.objects.all()

    if req_form:

        entries_obj = entries_obj.filter(form_id=req_form)

    if req_user:

        entries_obj = entries_obj.filter(user_id=req_user)



    if request.method=='POST':

        action = request.POST['action']



        if action == 'download_pdf':
        

            return JsonResponse({'reload':True})



        if action == 'move_trash':

            if request.user.type == 1  or request.user.can('can_trash_entry') :

                pass

            else:

                messages.warning(request, 'Permission denied...')

                return JsonResponse({'status': True})



            entry_id = int(request.POST['entry_id'])

            entry = Entries.objects.get(id=entry_id)



            cannot_delete_entries = user_form_permissions(request, 'cannot_add_delete_entries')

            role_cannot_delete_entries = role_form_permissions(request, 'cannot_add_delete_entries')

            if entry.form.id in cannot_delete_entries:

                messages.warning(request, 'You are not allowed to delete this entry.')

                return JsonResponse({'reload':True})



            if entry.form.id in role_cannot_delete_entries:

                messages.warning(request, 'You are not allowed to delete this entry.')

                return JsonResponse({'reload':True})



            Entries.objects.filter(id=entry_id).update(is_approved=0,is_trash=1)

            count = entries_obj.filter(is_trash=1).count()

            approved = entries_obj.filter(is_approved=1).count()

            unread = entries_obj.filter(is_read=0,is_trash=0).count()

            unapproved = entries_obj.filter(is_approved=0,is_trash=0).count()



            return JsonResponse({'status': True, 'message': 'Entry moved to trash!','trash':count,'entry_id':entry_id,'approved':approved,'unread':unread,'unapproved':unapproved})



        if action == 'restore_trash_item':

            if request.user.type == 1  or request.user.can('can_trash_entry') :

                pass

            else:

                messages.warning(request, 'Permission denied...')

                return JsonResponse({'status': True})



            entry_id = int(request.POST['restore_id'])

            Entries.objects.filter(id=entry_id).update(is_trash=0)

            count = entries_obj.filter(is_trash=1).count()

            approved = entries_obj.filter(is_approved=1).count()

            unread = entries_obj.filter(is_read=0,is_trash=0).count()

            unapproved = entries_obj.filter(is_approved=0,is_trash=0).count()

            return JsonResponse({'status': True, 'message': 'Entry restored !', 'trash':count,'entry_id':entry_id,'approved':approved,'unread':unread,'unapproved':unapproved})



        if action == 'create_duplicate':

            #check submission limits

            limit_info = check_submission_limits(request)

            if not limit_info['remaining_limit']:

                messages.warning(request, limit_info['msg'])

                return JsonResponse({'status': False})



            if request.user.type == 1  or request.user.can('can_duplicate_entry') :

                pass

            else:

                messages.warning(request, 'Permission denied...')

                return JsonResponse({'status': True})



            cannot_delete_entries = user_form_permissions(request, 'cannot_add_delete_entries')

            role_cannot_delete_entries = role_form_permissions(request, 'cannot_add_delete_entries')

            if request.user.type==3:

                if id in cannot_delete_entries or id in role_cannot_delete_entries:

                    messages.warning(request, 'You are not allowed to duplicate entry.')

                    return JsonResponse({'status': True})



            entry_id = int(request.POST['duplicate_id'])

            old_entry = Entries.objects.get(id=entry_id)

            d0 = datetime.now().date()

            d1 = datetime.date(old_entry.created_at)

            delta = d0 - d1



            now = timezone.now()

            last_populate_time = old_entry.created_at + timedelta(days=old_entry.form.delay_time_limit)



            if 'folder_id' in request.POST and 'duplicate_id' in request.POST:

                folder = request.POST.get('folder_id')

                details_entry = EntryDetails.objects.filter(entry=old_entry)

                old_entry.pk = None

                old_entry.save()

                entry_new = Entries.objects.all().order_by('-id')[0]

                for data in details_entry:

                    if data.field.not_populate_on_duplicate and data.entry.form.not_populate and now > last_populate_time:

                        data.value = ''

                    if data.field.not_populate_on_duplicate and data.entry.form.not_populate == False:

                        data.value = ''



                    EntryDetails.objects.create(entry=entry_new,field=data.field,value=data.value, info=data.info)

                FolderEntries.objects.create(entry=entry_new,folder=Folders.objects.get(id=folder))

                messages.success(request,'Duplicated folder entry has been created')

                return redirect('edit_entry',entry_id=entry_new.id)

            else:

                pass

            new_entry = Entries.objects.create(user=old_entry.user,form=old_entry.form)

            details_entry = EntryDetails.objects.filter(entry=entry_id)

            for data in details_entry:



                if data.field.not_populate_on_duplicate and data.entry.form.not_populate == False:

                    data.value = ''



                if data.field.not_populate_on_duplicate and data.entry.form.not_populate and now > last_populate_time:

                    data.value = ''





                EntryDetails.objects.create(entry=new_entry,field=data.field,value=data.value,info=data.info)

            

            #add submission details

            add_submission(request, new_entry, 'duplicate')          

            #send notification if submission reached 50 or 80 % 

            submission_limit_notification(request)



            messages.success(request,'Duplicated entry has been created')

            return redirect('edit_entry',entry_id=new_entry.id)



        if action == 'delete_entry':

            if request.user.type == 1  or request.user.can('can_trash_entry') :

                pass

            else:

                messages.warning(request, 'Permission denied...')

                return JsonResponse({'status': True})



            entry_id = int(request.POST['delete_id'])

            entry = Entries.objects.get(id=entry_id)



            cannot_delete_entries = user_form_permissions(request, 'cannot_add_delete_entries')

            role_cannot_delete_entries = role_form_permissions(request, 'cannot_add_delete_entries')

            if entry.form.id in cannot_delete_entries or entry.form.id in role_cannot_delete_entries:

                messages.warning(request, 'You are not allowed to delete this entry.')

                return JsonResponse({'reload':True})



            field_types = ['fileupload', 'signature']

            entry_details = EntryDetails.objects.filter(entry_id=entry.id, field__type__in=field_types)

            dir_path = settings.MEDIA_ROOT+'/form/'

            if request.session.get('domain', ''):

                dir_path = dir_path+request.session.get('domain')+'/'

            if entry_details:

                for data in entry_details:

                    if data.value and not data.value == 'None':

                        os.remove(dir_path+data.value)

                    if data.info:

                        extra_files = json.loads(data.info)

                        for file in extra_files:

                            os.remove(dir_path+file)



            Entries.objects.filter(id=entry_id).delete()

            return JsonResponse({'status': True, 'message': 'Entry deleted !!','entry_id':entry_id})



        if action == 'mark_as_read':

            if request.user.type == 1  or request.user.can('can_mark_entry_unread') :

                pass

            else:

                messages.warning(request, 'Permission denied...')

                return JsonResponse({'status': True})



            entry_id = int(request.POST['read_id'])

            Entries.objects.filter(id=entry_id).update(is_read=1)

            count = entries_obj.filter(is_read=0,is_trash=0).count()

            return JsonResponse({'status': True, 'message': 'Entry marked as read !!','count':count,'entry_id':entry_id})



        if action == 'mark_as_unread':

            if request.user.type == 1  or request.user.can('can_mark_entry_unread') :

                pass

            else:

                messages.warning(request, 'Permission denied...')

                return JsonResponse({'status': True})



            entry_id = int(request.POST['unread_id'])

            Entries.objects.filter(id=entry_id).update(is_read=0)

            count = entries_obj.filter(is_read=0,is_trash=0).count()

            return JsonResponse({'status': True, 'message': 'Entry marked as unread !!','count':count,'entry_id':entry_id})



        if action == 'disapprove_entry':

            if request.user.type == 1  or request.user.can('can_aprv_deny_entry') :     

                pass

            else:

                messages.warning(request, 'Permission denied...')

                return JsonResponse({'status': True})



            entry_id = int(request.POST['disapprove_id'])

            Entries.objects.filter(id=entry_id).update(is_approved=0)

            apprrove_count = entries_obj.filter(is_approved=1,is_trash=0).count()

            un_count = entries_obj.filter(is_approved=0,is_trash=0).count()

            return JsonResponse({'status': True, 'message': 'Entry disapproved !!','apprrove_count':apprrove_count,'entry_id':entry_id,'un_count':un_count})



        if action == 'approve_entry':


            if request.user.type == 1  or request.user.can('can_aprv_deny_entry') :

                pass

            else:

                messages.warning(request, 'Permission denied...')

                return JsonResponse({'status': True})



            entry_id = int(request.POST['approve_id'])

            entry = Entries.objects.get(id=entry_id)

            entry_user = User.objects.get(id=entry.user.id)

            Entries.objects.filter(id=entry_id).update(is_approved=1)

            apprrove_count = entries_obj.filter(is_approved=1,is_trash=0).count()

            un_count = entries_obj.filter(is_approved=0,is_trash=0).count()

            notify.send(sender=request.user, recipient=[entry_user], verb='An entry has been approved. Click to view entry.', data={"entry":entry_id, "view_entry":True})

            return JsonResponse({'status': True, 'message': 'Entry approved !!','apprrove_count':apprrove_count,'entry_id':entry_id,'un_count':un_count})



        if action == 'flag_entry':

            if request.user.type == 1  or request.user.can('can_flag_entry') :

                pass

            else:

                messages.warning(request, 'Permission denied...')

                return JsonResponse({'status': True})



            entry_id = int(request.POST['flag_id'])

            flag_to_user = request.POST.get('flag_to_user')

            flag_text = request.POST.get('flag_text')
            flagged_reason=request.POST.get('flag_reason')

            entry = Entries.objects.get(id=entry_id)

            entry.is_flagged = True

            entry.flagged_at = time.strftime("%Y-%m-%d %H:%M:%S", time.gmtime())

            entry.save()

            flagged = Entries.objects.filter(is_flagged=1,is_trash=0).count()

            EntryFlags.objects.create(entry=entry, flagged_by=request.user, flagged_to=User.objects.get(id=flag_to_user), flag_text=flag_text,flagged_reason=flagged_reason)

            notify.send(sender=request.user, recipient=[User.objects.get(id=flag_to_user)], verb='You have a flagged entry in waiting.', data={"entry":entry_id, "view_entry":True})

            flagged_user = User.objects.get(id=flag_to_user)

            t = threading.Thread(target=flag_entry_email, args=(request, flagged_user, entry,flagged_reason))

            t.setDaemon(True)

            t.start()

            return JsonResponse({'status': True, 'message': 'Entry set to flagged !!', 'entry_id':entry_id, 'flagged':flagged})



        if action == 'bulk_approve':


            if request.user.type == 1  or request.user.can('can_bulk_aprv_deny_entry') :

                pass

            else:

                messages.warning(request, 'Permission denied...')

                return JsonResponse({'status': True})

            entry_id = []
            folder_ids = []

            entry_folder_id = request.POST['approve_bulk'].split(',')

            for ids in entry_folder_id:
                if 'folder_' in ids:
                    spt = ids.split("_")
                    folder_ids.append(int(spt[1]))  
                else:
                    entry_id.append(int(ids))          
            
           

            Entries.objects.filter(id__in=entry_id).update(is_approved=1)

            messages.success(request, 'Bulk entries approved.')

            return JsonResponse({'status': True})



        if action == 'bulk_disapprove':

            if request.user.type == 1  or request.user.can('can_bulk_aprv_deny_entry') :

                pass

            else:

                messages.warning(request, 'Permission denied...')

                return JsonResponse({'status': True})


            entry_id = []
            folder_ids = []
            entry_folder_id = request.POST['disapprove_bulk'].split(',')

            for ids in entry_folder_id:
                if 'folder_' in ids:
                    spt = ids.split("_")
                    folder_ids.append(int(spt[1]))  
                else:
                    entry_id.append(int(ids)) 
            
            

            Entries.objects.filter(id__in=entry_id).update(is_approved=0)

            messages.success(request, 'Bulk entries disapproved.')

            return JsonResponse({'status': True})



        if action == 'add_to_folder':

            if request.user.type == 1  or request.user.can('can_bulk_add_to_folder_entries') :

                pass

            else:

                messages.warning(request, 'Permission denied...')

                return JsonResponse({'status': True})

            entry_ids = []
            folder_ids = []
            entry_folder_id = request.POST.get('entry_ids_add').split(',')            

            for ids in entry_folder_id:
                if 'folder_' in ids:
                    spt = ids.split("_")
                    folder_ids.append(int(spt[1]))     
                else:
                    entry_ids.append(int(ids))      


           
            folder = request.POST.get('folder')

            mvfolder = Folders.objects.get(id=folder)

            if request.user.type == 3 and mvfolder.user != request.user:

                if mvfolder.type_id==2:

                    user_capability = check_user_capability(request,'move',mvfolder)

                    role_capability = check_role_capability(request,'move',mvfolder)

                    if user_capability == None or role_capability == None:

                        return JsonResponse({'status':True,'message':'You cannot perform this operation.'})

                    else:

                        pass

            FolderEntries.objects.filter(entry_id__in=entry_ids).delete()

            for en in entry_ids:

                FolderEntries.objects.create(entry_id=en, folder_id=folder)    


            if folder_ids:
                for fid in folder_ids: 
                    folder = Folders.objects.get(id = int(fid))
                    folder.parent = mvfolder  
                    folder.save()            

            messages.success(request,'Entries added to folders successfully.')

            return JsonResponse({'status':True})



        if action == 'remove_from_folder':

            if request.user.type == 1  or request.user.can('can_bulk_remove_folder_entries') :

                pass

            else:

                messages.warning(request, 'Permission denied...')         

                return JsonResponse({'status': True})


            entry_ids = []
            folder_ids = []
            
            entry_folder_id = request.POST.get('entry_ids_remove').split(',')  

            for ids in entry_folder_id:
                if 'folder_' in ids:
                    spt = ids.split("_")
                    folder_ids.append(int(spt[1]))    
                else:
                    entry_ids.append(int(ids))           

            if request.user.type!=1:

                f_user_ids = []

                f_role_ids = []

                my_folders = Folders.objects.filter(user=request.user).order_by('parent')

                user_access = FolderUsersCapabilities.objects.filter(user=request.user)

                for f in user_access:

                    if f.move:

                        f_user_ids.append(f.folder.id)   

                r_access = FolderRolesCapabilities.objects.filter(role=request.user.user_role.all().first().role)

                for r in r_access:

                    if r.move:

                        f_role_ids.append(r.folder.id)

                shared_folders = Folders.objects.filter(id__in=list(set(f_user_ids).intersection(set(f_role_ids))), parent=None).order_by('parent')

                folders_list = my_folders | shared_folders

                for e in entry_ids:

                    fentry = FolderEntries.objects.get(entry_id=e)

                    if fentry.folder.id in folders_list:

                        pass

                    else:

                        fentry.delete()

                messages.warning(request, 'If any entries are not removed, then you may not have permissions !')     

            else:

                FolderEntries.objects.filter(entry_id__in=entry_ids).delete()


            messages.success(request,'Entries removed from folders successfully.')

            return JsonResponse({'status':True})



        if action == 'bulk_trash_entry':

            if request.user.type == 1  or request.user.can('can_bulk_trash_entries') :

                pass

            else:

                messages.warning(request, 'Permission denied...')

                return JsonResponse({'status': True})



            entry_ids = request.POST.get('entry_ids_remove').split(',')

            entries = Entries.objects.filter(id__in=entry_ids).update(is_trash=1)

            count = entries_obj.filter(is_trash=1).count()

            messages.success(request, 'Bulk entries moved to trash.')

            return JsonResponse({'status': True})



        if action == 'bulk_restore_entry':

            if request.user.type == 1  or request.user.can('can_bulk_restore_entries') :

                pass

            else:

                messages.warning(request, 'Permission denied...')

                return JsonResponse({'status': True})



            entry_ids = request.POST.get('entry_ids_remove').split(',')

            entries = Entries.objects.filter(id__in=entry_ids).update(is_trash=0)

            count = entries_obj.filter(is_trash=1).count()

            messages.success(request, 'Bulk entries restored.')

            return JsonResponse({'status': True})



        if action == 'bulk_delete_entry':

            if request.user.type == 1  or request.user.can('can_bulk_delete_entries') :

                pass

            else:

                messages.warning(request, 'Permission denied...')

                return JsonResponse({'status': True})



            entry_ids = request.POST.get('entry_ids_remove').split(',')

            field_types = ['fileupload', 'signature']

            dir_path = settings.MEDIA_ROOT+'/form/'

            if request.session.get('domain', ''):

                dir_path = dir_path+request.session.get('domain')+'/'

            entry_details = EntryDetails.objects.filter(entry_id__in=entry_ids, field__type__in=field_types)

            if entry_details:

                for data in entry_details:

                    if data.value and not data.value == 'None':

                        os.remove(dir_path+data.value)

                    if data.info:

                        extra_files = json.loads(data.info)

                        for file in extra_files:

                            os.remove(dir_path+file)

            Entries.objects.filter(id__in=entry_ids).delete()

            messages.success(request, 'Bulk entries deleted.')

            return JsonResponse({'status': True})



        if action == 'delete_file':

            entry_detail_id = int(request.POST['entry_detail_id'])

            file_name = request.POST['file_name']

            detail = EntryDetails.objects.get(id=entry_detail_id)

            files = []

            if detail.info:

                files = json.loads(detail.info)

            if file_name == detail.value:

                detail.value = None

                if files:

                    detail.value = files[0]

                    files.remove(detail.value)

            elif file_name in files:

                files.remove(file_name)



            if not files:

                detail.info = None

            else:

                detail.info = json.dumps(files)



            try:

                os.remove(settings.MEDIA_ROOT+'/form/'+file_name)

            except OSError:

                pass



            detail.save()

            return JsonResponse({'status': True})
